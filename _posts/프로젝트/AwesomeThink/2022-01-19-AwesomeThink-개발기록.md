---
title: AwesomeThink 앱 개발 일지 - 2
author:
name: 우영
link: https://github.com/Jang-zn
date: 2022-01-19
categories: [프로젝트, AwesomeThink]
tags: [프로젝트, AwesomeThink]
render_with_liquid: false
---

### 그냥 갖다 박으니까 한계가 온다

Provider 쓰다가 Stream쓰다가 아주 코드가 난장판이 되어버림

StreamBuilder 쓰다가 혈압올라서 결국 GetX로 상태관리 하는 방식으로 다 뒤집어 엎어버릴 예정

Future / async / await 내가 뭘 잘못쓰고 있는건지 계속 데이터 다 불러오기 전에 빌드되고 난리나서

이리저리 코드위치 바꿔보는데.. 이래선 안되겠다 싶다

안써봤고, 앞으로도 안쓸 Provider는 잠시 제쳐두고 회사 일에 쓸 GetX로 갈아엎어야겠다..




## 작업현황

<h3>1. 화면구성</h3>
  - 로그인페이지 完
  - 가입 페이지 完
  - 회원 메인페이지 레이아웃 完
  - 어드민 메인페이지 레이아웃 + drawer 작업 完
  - 어드민 가입승인 페이지 → 完
    <span style="color:red">- 회원 주간 근태 확인페이지 → GetX로 갈아엎고 진행</span><br>
    <span style="color:red">- 어드민 회원별 주간 근태 확인페이지 → GetX로 갈아엎고 진행</span><br>

<h3>2. 기능</h3>
  - firebase를 통한 가입/로그인 기능 完
  - 출/퇴근 기능 完
    <span style="color:red">- 멤버별 근태관련 DB 完 -- 뭔가 잘못돼서 뒤집어 엎고 다시 할 예정</span><br>
    <span style="color:red">- 휴무신청 기능 미완 - 이 망할것때문에 다 갈아엎게 생김</span><br>

<h3>3. 남은거</h3>
  - 주단위 근태확인기능 (달력)
  - admin 주간 근태현황 확인
  - admin 직원 근태 수정
  - admin 당일 출퇴근 확인


<br><br>
비동기처리 그냥 쓰면 됐던거같은데 돌아버리겠다 진짜


