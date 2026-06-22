# 🐧 Making Linux Commands with C

> C 언어로 **메모리 상의 가상 파일 시스템**을 구현하고, 그 위에서 동작하는 리눅스 셸 명령어(`ls`, `cd`, `cat`, `pwd`, `find`, `grep`, `chown` 등)를 직접 만든 팀 프로젝트입니다.
> 실제 OS의 파일 시스템을 건드리지 않고, **트리 자료구조로 디렉터리 구조를 시뮬레이션**하여 명령어의 내부 동작 원리를 학습하는 것을 목표로 했습니다.

![C](https://img.shields.io/badge/C-00599C?style=flat&logo=c&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![GCC](https://img.shields.io/badge/GCC-Compiler-blue?style=flat)

---

## 📌 한눈에 보기

| 항목 | 내용 |
|------|------|
| **프로젝트 유형** | 팀 프로젝트 (시스템 프로그래밍) |
| **사용 언어** | C |
| **핵심 개념** | 트리 자료구조(LCRS), 포인터, 동적 메모리, 문자열 파싱, REPL |
| **구현 명령어** | `ls` · `cd` · `cat` · `pwd` · `find` · `grep` · `chown` · `exit` |
| **빌드** | `gcc -o main *.c` |

---

## 👤 담당 역할 — 메인 시스템 설계 및 구현

> 본 저장소에서 **제가 맡은 부분은 프로젝트의 뼈대가 되는 "메인 시스템"** 입니다.
> 팀원들이 각 명령어를 독립적으로 구현할 수 있도록, **공통 자료구조와 명령어 분배 체계, 실행 루프**를 설계했습니다.

| 담당 파일 | 역할 |
|-----------|------|
| **`system.h` / `system.c`** | 파일 시스템의 핵심 자료구조(`tree`, `dir`, `file`) 설계 및 생성자 구현 |
| **`command.c`** | 사용자 입력을 파싱하고 각 명령어 함수로 분배하는 디스패처(`run()`) |
| **`main.c`** | 파일 시스템 초기화 및 셸 프롬프트(REPL) 루프 구현 |

> 즉, **"명령어를 담을 그릇(자료구조)"** 과 **"명령어를 받아 올바른 기능으로 연결하는 길(디스패처)"** 을 제가 책임졌고, 팀원들은 그 위에서 개별 명령어를 구현했습니다.

---

## 🌳 핵심 설계 — LCRS 트리로 표현한 파일 시스템

디렉터리는 자식 개수가 가변적이므로, **LCRS(Left-Child Right-Sibling)** 방식으로 트리를 구성했습니다.
하나의 디렉터리 노드는 *첫 번째 자식*과 *다음 형제*만 가리키면 되므로, 자식 수가 몇 개든 동일한 구조체로 표현할 수 있습니다.

```c
struct dir {
    char  name[20];
    int   permission;
    file* file_inside;     // 이 디렉터리가 보유한 파일 리스트(연결 리스트)
    struct dir* parent;    // 부모 디렉터리
    struct dir* left;      // 첫 번째 자식 디렉터리
    struct dir* right;     // 다음 형제 디렉터리
    char  user[10];
    struct tm* time;
};

struct tree {
    dir* root;       // 최상위 루트
    dir* current;    // 현재 작업 디렉터리 (cd로 이동)
};
```

**구조 예시** — `left`는 자식, `right`는 형제로 연결됩니다.

```
root
 └─(left)─ home ─(right)─ etc ─(right)─ var
              │
            (left)
              └─ ubuntu
```

> 📸 **이미지 영역 ①** — LCRS 트리 구조 다이어그램
> (디렉터리 트리에서 `left=자식 / right=형제 / parent=부모` 관계를 시각화한 그림)
> 권장 파일: `docs/lcrs-tree.png`
> ```markdown
> ![LCRS 트리 구조](docs/lcrs-tree.png)
> ```

이 설계 덕분에 `cd`, `find`, `pwd` 등 **경로를 따라 트리를 순회**하는 명령어들을 일관된 방식으로 구현할 수 있었습니다.

---

## 🔁 명령어 처리 흐름 (`command.c`의 `run()`)

`main.c`의 REPL 루프가 한 줄을 입력받으면 `run()`이 이를 토큰으로 나누고, 첫 토큰(명령어)에 따라 적절한 함수로 분배합니다.

```c
cmd = strtok(input, " ");          // 1) 명령어 추출

if (strcmp(cmd, "ls") == 0) {      // 2) 명령어별 분기
    opt = strtok(NULL, " ");
    lsCommand(structure->current, opt);
}
else if (strcmp(cmd, "cd") == 0) {
    run_cd(strtok(NULL, " "));
}
// ... grep, find, cat, pwd, chown, exit
else {
    printf("Unknown command: %s\n", cmd);
}
```

**처리 순서**

```
[ main.c ]  프롬프트 출력 → fgets로 한 줄 입력
     │
     ▼
[ command.c: run() ]  개행 제거 → strtok로 토큰화 → 명령어 분기
     │
     ▼
[ 각 명령어 함수 ]  lsCommand / run_cd / cat / grep / find / pwd / chown ...
```

> 📸 **이미지 영역 ② (선택)** — 명령어 디스패치 흐름도
> 권장 파일: `docs/command-flow.png`

---

## ✨ 구현된 명령어

| 명령어 | 설명 |
|--------|------|
| `ls` | 현재 디렉터리의 파일/디렉터리 목록 출력 (옵션 지원) |
| `cd` | 디렉터리 이동 — `..`(상위), `~`(홈), 경로 지정 지원 |
| `cat` | 파일 내용 출력 / `cat > file`(작성), `cat -n`(줄 번호) 지원 |
| `pwd` | 스택을 활용해 현재 경로를 루트부터 역추적하여 출력 |
| `find` | 트리를 순회하며 파일/디렉터리 탐색 |
| `grep` | 파일 내용에서 특정 패턴 검색 |
| `chown` | 파일/디렉터리의 소유자 변경 |
| `exit` | 셸 종료 |

> 📸 **이미지 영역 ③** — 실제 실행 화면 (터미널에서 `ls`, `cd`, `cat` 등을 입력하고 출력되는 모습)
> 권장 파일: `docs/demo.png` 또는 `docs/demo.gif`

> 💡 명령어 입력 시 프롬프트는 `osmanager:<현재디렉터리>$` 형태로 표시됩니다.

---

## 📂 프로젝트 구조

```
Making-Linux-Commands-with-C/
├── main.c            # [메인 시스템] 엔트리 포인트 + REPL 루프
├── command.c/.h      # [메인 시스템] 입력 파싱 및 명령어 디스패처
├── system.c/.h       # [메인 시스템] tree/dir/file 자료구조 및 생성자
├── getpointer.c/.h   # 경로 문자열 → 디렉터리 포인터 해석 (/, ~, ., ..)
├── ls.c/.h           # ls 명령어
├── cd.h              # cd 명령어
├── cat.h             # cat 명령어 (출력/작성/줄번호)
├── pwd.c/.h          # pwd 명령어
├── find.c/.h         # find 명령어
├── grep.c/.h         # grep 명령어
└── chown.c/.h        # chown 명령어
```

> 📝 참고: 저장소에 `src/`, `include/` 등 개발 과정에서 생긴 이전 버전·중복 파일과 컴파일 산출물(`main`, `*.swp`, `*_output.txt`)이 함께 포함되어 있습니다. 실제 빌드 대상은 **루트 디렉터리의 `.c` 파일들**입니다. (공개 전 정리 권장 — 아래 "개선할 점" 참고)

---

## 🚀 빌드 및 실행

```bash
# 1. 저장소 클론
git clone https://github.com/<your-username>/Making-Linux-Commands-with-C.git
cd Making-Linux-Commands-with-C

# 2. 컴파일 (루트의 모든 .c 파일을 함께 빌드)
gcc -o main *.c

# 3. 실행
./main
```

실행하면 가상 파일 시스템이 초기화되고, 셸 프롬프트가 나타납니다.

```
osmanager:root$ ls
osmanager:root$ cd testdir
osmanager:testdir$ cat testfile.txt
osmanager:testdir$ exit
```

---

## 🧠 배운 점

> ✍️ **작성 영역** — 아래는 예시입니다. 본인의 경험에 맞게 다듬어 주세요.

- **자료구조 설계가 협업의 기준이 된다**는 점을 체감했습니다. 트리·구조체를 먼저 명확히 정의해 두니, 팀원들이 각 명령어를 독립적으로 구현하기 수월했습니다.
- LCRS 표현을 직접 코드로 옮기며 **트리를 가변 자식 구조로 다루는 방법**을 익혔습니다.
- 포인터와 동적 메모리, 문자열 파싱(`strtok`)을 활용해 **하나의 셸 실행 루프(REPL)** 가 동작하도록 통합하는 경험을 했습니다.

---

## 🔧 개선할 점 (회고)

> ✍️ **작성 영역** — 포트폴리오에 솔직한 회고를 담으면 좋습니다. 예시:

- 저장소 내 중복 디렉터리(`src/`, `include/`)와 산출물 정리 → 빌드 구조 명확화
- `Makefile` 도입으로 빌드 자동화
- 명령어별 예외 처리 및 메모리 해제(`free`) 보강

---

## 👥 팀 구성

> ✍️ **작성 영역** — 팀원과 담당 명령어를 표로 정리하면 기여도가 명확해집니다.

| 이름 | 담당 |
|------|------|
| **본인** | 메인 시스템 (자료구조 · `command.c` · `main.c`) |
| 팀원 A | (예: ls, chown) |
| 팀원 B | (예: cat, cd) |
| 팀원 C | (예: find, grep, pwd) |