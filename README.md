# 리눅스 명령어 C 구현

리눅스에서 매일 쓰는 `ls`, `cd`, `cat` 같은 명령어를 C 언어로 직접 만들어 하나의 셸(명령어 입력 창)로 묶은 팀 프로젝트입니다.

실제 컴퓨터의 파일을 건드리지 않습니다. 프로그램이 메모리 안에 가상의 폴더와 파일을 만들고, 모든 명령어가 그 안에서 동작합니다. 명령어가 내부에서 어떻게 움직이는지 직접 만들어 보며 이해하는 것이 목표였습니다.

## 한눈에 보기

| 항목 | 내용 |
|------|------|
| 유형 | 운영체제 수업 팀 프로젝트 (5인) |
| 기간 | 2025.05 ~ 2025.06 |
| 언어와 도구 | C, gcc, pthread |
| 개발 환경 | AWS EC2 리눅스 서버에 팀이 함께 접속해 개발 |
| 셸에서 쓸 수 있는 명령어 | `ls`, `cd`, `cat`, `pwd`, `find`, `grep`, `chown`, `exit` |
| 단독 실행 프로그램 | `mkdir`, `rm` |

이 저장소에는 2025년 5월 16일에 팀원들의 코드를 하나로 합친 시점의 코드가 들어 있습니다.

## 어떻게 동작하나요

프로그램을 실행하면 아래 세 단계가 반복됩니다.

```
1. 입력   main.c       프롬프트를 띄우고 한 줄을 입력받습니다
2. 분배   command.c    첫 단어(명령어 이름)를 보고 맞는 함수를 부릅니다
3. 실행   ls.c 등      가상 파일 시스템을 읽거나 고친 뒤 결과를 출력합니다
```

명령어 하나가 함수 하나로 되어 있고, 셸은 이름에 맞는 함수만 불러 줍니다. 그래서 팀원이 각자 명령어를 따로 만든 뒤 `command.c`에 한 갈래씩 연결하는 방식으로 합칠 수 있었습니다.

## 가상 파일 시스템

폴더와 파일을 나무 모양(트리)으로 연결해 표현했습니다. `system.h`에 정의되어 있습니다.

- 폴더 하나는 부모 폴더, 첫 번째 하위 폴더, 옆 폴더, 자기 안의 파일 목록을 가리킵니다.
- 파일은 이름, 내용, 권한, 소유자, 만든 시각을 가집니다.
- 트리 전체는 최상위 폴더와 "지금 있는 폴더"를 기억합니다. `cd`는 이 "지금 있는 폴더"만 바꿉니다.

```c
struct dir {
    char name[20];
    int permission;
    file* file_inside;     // 이 폴더 안의 파일 목록
    struct tm* time;
    struct dir* parent;    // 부모 폴더
    struct dir* left;      // 첫 번째 하위 폴더
    struct dir* right;     // 옆 폴더
    char user[10];
};
```

하위 폴더가 몇 개든 "첫 번째 하위 폴더"와 "옆 폴더" 두 가지만 가리키면 되므로 폴더 수에 제한이 없습니다.

```
root
 └─ home ── etc ── var      (옆으로 이어진 폴더들)
      └─ ubuntu             (home의 첫 번째 하위 폴더)
```

`/home/ubuntu`나 `../docs` 같은 경로 글자를 실제 폴더 위치로 바꿔 주는 기능은 `getpointer.c`에 한 번만 만들어 두고 `ls`, `cd`, `chown`이 함께 씁니다.

## 구현한 명령어

셸 안에서 쓰는 명령어입니다.

| 명령어 | 하는 일 | 지원하는 사용법 |
|--------|---------|-----------------|
| `ls` | 폴더 안의 파일과 폴더 목록 출력 | `-a` 숨김 항목 포함, `-l` 권한과 소유자와 시각까지 자세히, 경로 지정 |
| `cd` | 폴더 이동 | `..` 상위 폴더, `~` 홈, 절대 경로와 상대 경로, `--help` |
| `cat` | 파일 내용 출력과 작성 | `cat 파일`, `cat -n 파일` 줄 번호, `cat > 파일` 새 파일 작성 (`:q` 입력으로 종료) |
| `pwd` | 현재 위치 출력 | 스택에 폴더 이름을 쌓아 최상위부터 차례로 출력 |
| `grep` | 파일에서 글자가 들어간 줄 찾기 | `grep 찾을글자 파일`, 결과를 화면과 `grep_output.txt`에 기록 |
| `find` | 파일과 폴더 목록 탐색 | 하위 폴더마다 스레드를 하나씩 만들어 탐색하고 `find_output.txt`에 기록 |
| `chown` | 파일이나 폴더의 소유자 변경 | `chown 새소유자 경로` |
| `exit` | 셸 종료 | |

`src/` 폴더의 `mkdir.c`와 `rm.c`는 셸에 연결되기 전 단계로, 각각 따로 실행하는 프로그램입니다.

| 프로그램 | 하는 일 | 지원하는 사용법 |
|----------|---------|-----------------|
| `mkdir` | 폴더 생성 | `-p`, `-m 권한`, 여러 폴더를 한 번에 주면 폴더마다 스레드를 만들어 동시에 생성 |
| `rm` | 파일과 폴더 삭제 | `-r` 하위 포함, `-f` 오류 무시, 여러 대상을 스레드로 동시에 삭제 |

## 담당 역할

제가 맡은 부분은 팀원들의 명령어가 올라가는 바탕입니다.

| 파일 | 내용 |
|------|------|
| `system.h`, `system.c` | 가상 파일 시스템의 폴더, 파일, 트리 구조와 이를 만드는 함수 |
| `command.c` | 입력을 단어로 나누고 명령어 이름에 맞는 함수로 넘기는 부분 |
| `main.c` | 파일 시스템을 준비하고 입력을 계속 받는 반복 구조 |
| `pwd.c` | `pwd` 명령어와 여기에 쓰는 스택 |

여기에 더해 다섯 명이 각자 만든 코드를 하나의 프로그램으로 합치고 실행해 확인하는 일을 했습니다. 나머지 명령어는 팀원 네 명이 나누어 구현했습니다.

## 빌드와 실행

리눅스와 gcc가 필요합니다.

```bash
git clone https://github.com/JiwonXD/Making-Linux-Commands-with-C.git
cd Making-Linux-Commands-with-C

# 셸 빌드 (최상위 폴더의 .c 파일을 함께 컴파일)
gcc -o main *.c

# 실행
./main
```

실행하면 연습용 파일 `testfile.txt`와 폴더 `testdir`가 들어 있는 가상 파일 시스템이 만들어지고 프롬프트가 나타납니다. 아래는 실제 실행 결과입니다.

```
osmanager:root$ ls
testfile.txt  testdir/
osmanager:root$ ls -l
total 10
-rwx------ 1 user group   1024 Nov 17 12:38 testfile.txt
drwxr-xr-x 2 user group   4096 Nov 17 12:38 testdir/
osmanager:root$ cat -n testfile.txt
	1  abcabcdabcde
osmanager:root$ grep abc testfile.txt
testfile.txt: abcabcdabcde
osmanager:root$ chown jiwon testfile.txt
Changed owner of file 'testfile.txt' to 'jiwon'
osmanager:root$ cd testdir
osmanager:testdir$ ls
testfile.txt
osmanager:testdir$ exit
```

`mkdir`와 `rm`은 따로 빌드합니다.

```bash
gcc -o mkdir src/mkdir.c -lpthread
./mkdir -p a/b c

gcc -o rm src/rm.c -lpthread
./rm -r /home/user/docs
```

## 폴더 구성

```
Making-Linux-Commands-with-C/
├── main.c            시작점, 입력을 계속 받는 반복 구조
├── command.c/.h      입력을 나누고 명령어 함수로 분배
├── system.c/.h       폴더, 파일, 트리 구조와 스택
├── getpointer.c/.h   경로 글자를 폴더 위치로 변환
├── ls.c/.h           ls
├── cd.h              cd
├── cat.h             cat
├── pwd.c/.h          pwd
├── find.c/.h         find
├── grep.c/.h         grep
├── chown.c/.h        chown
└── src/
    ├── mkdir.c       mkdir (단독 실행)
    └── rm.c          rm (단독 실행)
```

`src/`와 `include/`의 나머지 파일은 개발 중에 쓰던 이전 버전입니다. 셸은 최상위 폴더의 파일로 빌드합니다.
