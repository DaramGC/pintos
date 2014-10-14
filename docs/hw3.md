Pintos 3: 프로세스 계층 구조
=========================
2013011037 류형욱

과제의 목표 및 요구사항
-------------------
프로세스 계층 구조를 구현합니다. 사용자 프로세스에서 wait 및 exec 시스템 콜이 사양에 맞게 동작되도록 수정합니다. Pintos 문서에 따르면, wait 및 exec 시스템 콜은 다음과 같이 동작되어야 합니다.

### wait

wait 시스템 콜은 주어진 pid에 해당하는 프로세스가 실행 중이라면 프로세스가 종료될 때까지 반환되지 않습니다. 해당 프로세스가 정상적으로 종료되었다면 종료 상태(exit status)를 반환하며, 예외 등의 이유로 커널에 의하여 강제로 종료되었다면 -1을 반환합니다. pid에 해당하는 프로세스가 종료된 이후에 wait를 호출하는 것이 가능하며, 그 경우에도 종료 상태는 제대로 반환되어야 합니다.

wait 시스템 콜이 실패하고 -1을 반환하는 다른 몇 가지 경우도 있습니다. pid가 유효하지 않은 경우, 실패합니다. wait를 호출한 프로세스가 exec를 성공적으로 실행하고 얻은, 직접적인 한 단계 자식의 pid만이 유효합니다. 같은 pid로 두 번 이상 wait를 수행하는 경우 두 번째부터는 실패합니다.

여러 자식 프로세스가 생성된 이후 여러 자식 프로세스가 종료되는 일, 부모 프로세스가 여러 자식 프로세스에 대한 wait를 호출하는 일은 어떤 순서로도 일어날 수 있습니다. 또 부모 프로세스가 자식 프로세스를 wait하지 않고 스스로 종료될 수 있습니다.

프로세스와 관련된 리소스는 항상 안전하게 해제되어야 합니다.

### exec

주어진 명령줄로 새 프로세스를 실행하고 pid를 반환합니다. 성공적인 수행의 결과로 pid를 얻지 못한 다른 모든 경우 -1을 반환해야 합니다. exec가 반환될 때 pid는 유효하거나 그렇지 않은 것으로 결정되어야 하기 때문에, 프로그램이 메모리에 적재되기 전까지 이 함수는 반환되지 않습니다.

문제를 해결하기 위한 Pintos 분석 내용
--------------------------------
- 요구사항을 구현하는 과정에서 바쁜 대기를 사용하는 방법은 고려하지 않습니다.
- 새로운 프로세스가 생성되었을 때부터 프로그램을 메모리에 적재하는 작업이 완료될 때까지 부모 프로세스를 블록된 상태로 두기 위해 동기화해야 합니다. 새로운 스레드에서 수행할 프로그램을 적재하는 작업은 새로운 스레드에서 일어납니다.
- 추가적으로 두 개의 동기화 지점을 사용하여 wait 시스템 콜의 요구사항을 만족할 수 있습니다.
- 부모 프로세스가 wait 함수에서 자식 프로세스의 종료 여부를 확인하고, 자식 프로세스가 자신이 종료되었음을 알려주기 위한 동기화가 필요합니다. 
- 부모 프로세스가 자식 프로세스에 대하여 첫 번째 wait 시스템 콜을 마쳤거나, 부모 프로세스가 자식 프로세스에 대하여 wait 없이 종료되었을 때, 그때 정확하게 자식 프로세스의 스레드 구조를 해제하기 위한 동기화가 필요합니다.
- 부모 프로세스가 하나의 자식 프로세스에 대하여 두 번 이상 wait를 수행할 경우 두 번째부터는 실패해야 하는 명세에 따라서, 첫 번째 wait 시스템 콜이 완료되었을 때 해당 자식 프로세스에 대한 정보를 해제할 수 있습니다. 자식 프로세스가 없는 것과 같이 wait 함수에서 -1을 반환해야 하는 다양한 경우를 구분하지 않고 처리할 수 있습니다.
- 부모가 자식 프로세스에 대하여 wait 없이 종료되었을 때는 종료될 때 wait를 수행하지 않아 자식 프로세스 리스트에 여전히 저장되어 있는 모든 프로세스의 파괴 세마포어를 하나 증가시킵니다. 결과적으로 자식 프로세스는 부모 프로세스과 상관 없이 종료됩니다.
- Pintos에서 제공하는 세마포어와 이중 연결 리스트 구현을 바로 사용할 수 있습니다.

해결 과정
-------
1. 스레드가 자식 스레드를 이중 연결 리스트로 관리할 수 있도록 합니다. 스레드 구조체에 종료 상태, 적재 성공 여부, 스레드 종료 대기 세마포어, 스레드 파괴 대기 세마포어, 스레드 적재 대기 세마포어를 추가합니다.
1. exit 시스템 콜이 종료 상태를 저장하도록 합니다.
1. 스레드가 초기화될 때 새롭게 추가한 세마포어와 리스트를 추가적으로 초기화하도록 합니다. 자식 스레드를 생성할 때 자신의 리스트에 자식 스레드를 추가하도록 합니다. 자식 스레드 중에서 tid가 일치하는 스레드를 찾는 함수 thread_get_child를 구현합니다.
1. 적재 대기 세마포어를 이용하여 exec 함수가 의도한 대로 동작하도록 합니다.
1. wait 함수가 의도한 대로 동작하도록 하기 위하여 여러 함수가 종료 대기 세마포어와 파괴 대기 세마포어를 사용하도록 수정합니다. 자세한 내용은 소스 코드 주석으로 설명하였습니다.

결과
---
모든 목표를 달성하였습니다. 추가적으로, 시스템 콜을 호출할 때 주소의 범위 검사와 더불어 올바른 페이지에 대한 가상 메모리 주소인지를 추가적으로 검사하도록 했습니다. 결과적으로, userprog의 args\*, sc\*, halt, exit, create\*, write-stdin, write-bad-fd, exec\*, wait\*, multi-recurse 테스트를 통과합니다.