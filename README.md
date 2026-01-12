# sveltekit-brainkit
Agent rules for sveltekit project

# Porting prompt
 rules-nextjs 에 있는 frontend-architecture 규칙을 sveltekit v5로 포팅한 뒤 rules 폴더에 다음과 같은 포맷으로 포팅해줘

 1. 파일 구조는 다음과 같다.
 rules/<rule name>.md 
 rules/<rule name>/references/<detail>.md

 2. 전체 파일 길이가 엄청 긴게 아니라면 하나의 파일로 merge 해도 된다. (reference 없이) 
 3. 해더는 다음과 같은 포맷으로 자것ㅇ해줘 

 ---
trigger: always_on
glob: "*"
description: "Backend Architecture Rules"
---

 4. 인용 방식은 <rule name>.md 기준 상대경로로 작성해줘 예를들면

 See [Workspace Structure](backend-architecture/references/workspace_structure.md) for Monorepo layout.

 과 같다. 