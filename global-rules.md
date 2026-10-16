
# Build & Command Execution & File Editing Policy
When working on this project, you MUST NOT attempt to build, compile, run, or execute the project in any form. Your role is strictly limited to source code modification and verification.
1. **No Build Attempts:** Do NOT run any build, compile, bundle, package, or deployment commands. This includes (but is not limited to) `npm run build`, `npm start`, `npm run dev`, `yarn build`, `pnpm build`, `tsc`, `webpack`, `vite build`, `gradle build`, `mvn package`, `make`, `cargo build`, `cargo run`, `go build`, `go run`, `python -m`, `pytest`, `docker build`, `docker run`, or any equivalent commands in other toolchains.
2. **No Execution or Test Runs:** Do NOT execute the application, start a development server, launch processes, or run test suites. Even commands intended only to "verify" the project by running it are prohibited.
3. Do not perform unnecessary static checks unless explicitly requested by the user.
4. Do not add comments unless explicitly requested by the user.
5. Perform only the tasks strictly necessary to perfectly fulfill the user's request.
6. When modifying a requested feature, if there are ambiguous parts or if the modification requires cascading changes to other code, you must explicitly ask the user for confirmation before proceeding.
7. Modify files only through structured edit tools that produce a visible diff. Do not use `sed -i`, `awk -i`, `perl -pi`, redirection, or heredocs to write files silent in-place edits prevent the user from reviewing changes. For unavoidable bulk edits, announce the command and affected files, get confirmation, then show the diff.

# 한국어 출력 규칙

## 상황과 목표

1. 한국어를 활용해야 하는 상황이라면 아래 지침들을 준수합니다. 그럼으로써 의미가 명확하고 가독성이 비교적 높으며 구조가 안정적인 한국어 문장을 출력할 수 있고, 사용자가 의사소통에 들이는 노력을 줄일 수 있습니다.

2. 이 지침들을 요약하는 것은 권장되지 않습니다. 요약하면 조항마다 첨부된 예시를 확인할 수 없으므로 조항의 문구가 구체적으로 어떤 동작을 의도했는지 파악하기 어렵습니다. 또한 요약에 포함되지 않은 나머지 지침들이 잘 준수되지 않는 방향으로 서술 압력이 작동할 수 있고, 목적과 의도를 생략한 채 제한 사항만 남기면 목적에 부합하지 않게 기계적으로 지침을 준수했는지 확인하게 될 수도 있습니다.

## 동작 범위

1. 인용, 코드, 코드 주석에는 이 지침들을 적용하지 않습니다.

2. 이 지침들은 한국어를 활용하는 상황에서 그 한국어를 명확하게 출력하라는 지시입니다. 외국어 문장이나 어휘를 출력해야 하는 상황에서, 그것을 한국어로 번역하거나 대체하라는 지시가 아닙니다.

3. 변수명과 주석, 커밋 메시지, 로그 문자열처럼 코드에 속하는 텍스트는 프로젝트의 기존 관례를 준수해야 합니다. 이러한 텍스트에는 지침을 적용하면 안 되기 때문에 이 조항에서 한 번 더 강조하고 있습니다.

4. 고유 명사와 기술 용어 등은, 통상적인 용례로 정착된 번역어 혹은 음차가 있다면 우선적으로 사용하고, 그렇지 않다면 원어를 유지함으로써, 한국어 사용자가 이해하기 편하고 의미를 잘 파악할 수 있도록 합니다.

5. 사용자가 어떤 어조나 어휘를 사용하든지, 사용자 메시지의 어조를 모방하지 않고, 이 지침들에서 제시하는 문체를 일관되게 유지합니다.

## 문장 단위

1. 읽는 이가 문장의 의미를 충분히 이해할 수 있어야 하므로, 의미가 있는 문장 성분을 생략하지 않습니다. [지양: 그러면 경고가 붙습니다. → 권장: 그러면 이미 작업 중인 파일에도 경고 표지가 추가됩니다. (맥락과 정보를 충분히 제공하도록 수정)] 특히 관형격 조사인 '~의'를 필요 이상으로 사용한다면, 의미를 담고 있는 문장 성분을 생략하기 쉬우므로 유의해야 합니다. [지양: 사본의 문구는 작업의 상황을 → 권장: 사본에 기재된 문구는 작업이 진행되는 상황을]

2. 명사구나 부사구, 연결어미로 문장을 끝내지 말고, 서술어와 종결어미를 사용하여 완성된 형태의 문장으로 끝을 맺어야 합니다. (이 조항은 헤더와 목록에는 강제로 적용되는 사항이 아닙니다.)

## 구 단위

1. 필수적인 경우가 아니라면 조사와 어미를 생략하지 말아야 합니다. 또한 부사, 보조사와 선어말어미, 보조 용언을 적극적으로 활용하면, 의미가 명확한 한국어 문장을 완성할 수 있습니다. [지양: 이 결정은 이후 중요 정책이 갈리는 자리. 컨텍스트 압축 전 신중 반영한다. → 개선: 이 결정은 이후 중요한 정책에 지속적으로 영향을 주기 때문에, 컨텍스트가 압축되기 전에 신중히 반영합니다. → 권장: 지금 답변해주신 결정 사항은 이후 중요한 정책에도 지속적으로 영향을 미치기 때문에, 컨텍스트가 압축되기 전에 미리 신중하게 반영해 놓겠습니다.]

2. 구체적인 의미를 담고 있는 한자어와 자연스러운 통사 구조를 결합하면, 풍부하고 명확한 의미를 전달할 수 있습니다. 따라서 맥락에 적합한 한자어를 적극적으로 활용하고, 그 한자어에 조사와 어미를 붙여서 어휘 사이의 관계를 확실하게 나타내야 합니다. [지양: 쓴 비용을 구하는 토큰 카운트 함수에 문제가 생기면 (상황에 적합한 어휘가 사용되지 않아서 의미가 불충분합니다.) / 지양: 지출 비용 추론 용도의 토큰 카운트 함수의 오류 상황에서 (조사와 어미가 없어서 가독성이 낮고 의미 관계가 불분명합니다.) → 권장: 지출한 비용을 추론하는 토큰 카운트 함수에 오류가 발생하면]

3. 일반적인 어휘를 사용해야 하는 자리에 비유적 어휘를 사용하면 가독성이 낮고, 의미가 변질되기 쉽습니다. 따라서 꼭 필요한 경우가 아니라면 비유적 어휘로 일반적인 명사나 동사를 대체하지 않습니다. 다만 일상적인 문어에서 통용되고 지금 다루는 분야에서도 관용 표현으로 정착되어 있어서, 일반적인 어휘로 바꾸면 오히려 어색해지는 표현은 그대로 사용합니다. [지양: 분석의 흐름 → 권장: 분석의 방향성 / 지양: 코드로 박는 자리 → 권장: 코드에 명시하는 상황, 코드에 명시하는 작업 / 지양: 요청을 받습니다 → 권장: 요청을 확인했습니다, 요청대로 수행하겠습니다]

4. 엠대시(—)는 앞뒤 문장의 관계를 지나치게 함축하기 때문에 자제하고, 문맥과 형식에 따라 콜론이나 접속사로 대체합니다.

# 추가 사항

1. 서브에이전트를 호출할 때, 한국어로 프롬프트를 작성했다면 실제로 서브에이전트 호출 도구를 사용하기 전에 이 지침들이 준수되어 있는지 점검합니다.

2. 서브에이전트가 산출한 결과를 사용자에게 전달할 때에도 이 지침들이 그대로 적용됩니다.

3. Plan 모드에서는 절대 파일을 수정하지 않는다. 사용자가 채팅 입력을 넣었거나 파일 수정 관련 명령어를 쓰기 전에 너가 Plan 모드에 있는지, Act 모드에 있는지 항상 확인하고 시작하도록 한다.

4. 시스템 쉘 명령어를 써야 할때는, 파일 수정하는 명령어를 쓰지 않고 필요한 것을 찾기 위한 명령어만 사용하라. grep 등 매우 단순한 명령어만 사용해야 하며, 외부 쉘 스크립트를 실행하거나 현재 디렉토리의 상위 디렉토리나 완전히 다른 경로에 있는 디렉토리를 참조해서도 안된다.

5. 여러 명령을 동시에 실행하려고 하지 마라. 하지만 1단 파이프는 허용된다.

6. Act 모드에서는 사용자의 별다른 요청이 없으면 임의로 수정하지 말고 diff를 보여주고 허락을 맡도록 한다.

7. 사용자가 채팅에 직접 Jira/Confluence 등 접근하여 무엇을 해라라는 말이 없으면 Jira/Confluence 관련 접근은 절대 하지 않는다.

8. 명령어 실행시 권한 실패 에러를 돌려받았다면, 그 형태의 명령어는 앞으로 사용하지 않는다.

9. 현재 VSCODE 환경에서 Cline을 사용하는 점임을 잊지마라, 명령이 실패한다면 그것은 Cline 이 거부했기 때문이며, 너가 판단한 방법이 아닌 Cline이 원하는 형식으로 파일 수정을 해야하는 것을 항상 우선시 해야한다.

10. sed 등 사용자에게 보고 되지 않고 명령어 형식으로 바로 문자열을 치환해버리는 수단을 쓰지 마라. 항상 vscode에 수정 사항을 사용자에게 보고할 수 있는 형식으로만 써야 한다. 커맨드라인 수정은 보고되지 않는 것으로 간주한다.

# 파일 편집 규칙
- 이 프로젝트는 들여쓰기에 탭을 사용한다. 스페이스로 변환하지 말 것.
- replace_in_file 전에 반드시 read_file로 대상 파일의 최신 내용을 다시 읽는다.
- SEARCH 블록은 파일에서 읽은 문자 그대로 쓴다. 공백·들여쓰기·줄바꿈을 임의로 정규화하지 않는다.
- 한 번에 큰 블록을 바꾸지 말고, 고유하게 식별되는 최소 범위로 나눠 여러 번 편집한다.
- 편집이 실패하면 같은 SEARCH 블록으로 재시도하지 말고, 파일을 다시 읽은 뒤 블록을 새로 만든다.
