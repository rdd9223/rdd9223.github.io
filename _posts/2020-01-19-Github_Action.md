---
title: "Github Action으로 자동 test하기!"
excerpt: "Node.js + Jest + Github Action"
toc: true
toc_sticky: true
header:
  teaser: /assets/images/bio-photo-keyboard-teaser.jpg

categories:
  - Github Action
tags:
  - Github Action
  - CI
last_modified_at: 2020-11-29T21:00:00-05:00
---

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cc912848-4d20-4731-aacb-9b9fcb1b10c0/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cc912848-4d20-4731-aacb-9b9fcb1b10c0/Untitled.png)

## 개요

---

CI (Continuous Integration)를 위해 Jenckins나 써클 CI를 사용하지 않고, Github에 내장된 Action기능을 사용해서 PR시 Test Script를 자동 실행하고, 오류가 있다면 PR을 close하는 action을 만들어 봤습니다!

## 개념

---

- Microsoft사의 Azure의 가상머신을 이용해 GithubAction을 등록하면 안의 스크립트가 가상머신 안에서 돌아간다. 이를 통해 Github와 연동해서 다양한 이벤트를 줄 수 있다.
- 여기서 사용되는 가상머신의 스펙은 2-Core, 7GB RAM, 14GB SSD의 스펙을 가지고 있다. 좋은 스펙은 아니라 조금 느리다.
- 공개 레포지토리는 무료이고, private 레포지토리는 어느정도 지나면 과금이 된다. 테스트와 린트정도만 체크하는 용도면 어느정도 여유가 있다.
  무료 한계는 500MB Storage, 2,000Minites (per month)이다.
- Github 레포지토리의 Action항목을 클릭해서 만들어도 되고, 프로젝트의 `./github/workflows/~~~.yml` 으로 만들어도 된다. 푸시를 하게되어서 원격 저장소에 저장되면, 그 때부터 적용된다. `~~~(파일명)`은 아무렇게나 해도 된다!

## 전체 코드

---

```yaml
# Action명, 커스텀 가능하다
name: Node.js CI

# 이벤트 감지
# 아래의 명령어는 모든 브랜치에 한해서 PR감지 (브랜치 지정할 수 있음)
on: pull_request

# 작업단위. 실질적인 github action이 실행되는 곳
jobs:
  build:  # 작업의 이름. 커스텀 가능하다.
    runs-on: ubuntu-18.04  # 가상머신 OS 설정. 버전설정, window, MacOS도 사용할 수 있다.
    services:  # 서비스할 목록들 (ex. Redis, Postgresql ...)
      mysql:   # 사용할 서비스 명
        image: mysql:5.7  # 사용할 서비스의 이미지(도커 개념 기반)
        env:   # 사용할 서비스의 환경설정
          MYSQL_USER: test
          MYSQL_PASSWORD: test
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: PLAM
        ports: # 열어줄 포트, 8080:3306 형태로도 가능
          - 3306
        options: >-  # 서비스에 걸어줄 옵션
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    strategy:  # 가상머신 안에서 사용할 작업환경
      matrix:
        node-version: [12.x]  # node 12버전을 적용해서 action (타버전 적용가능)
    steps:  # 실질적인 작업 순서를 정하는 곳
      - uses: actions/checkout@v2  # 사용할 액션을 정의하는 곳
      - name: Use Node.js ${{ matrix.node-version }} # 사용할 액션의 이름을 정의
        uses: actions/setup-node@v1 # node 액션을 사용할 것이다.
        with:  # 액션에 필요한 파라미터들을 쓰는 곳
          node-version: ${{ matrix.node-version }} # 위의 strategy에 적용한 노드의 버전을 액션에서 가져온다.
      - name: verify MySQL connection from host
        # 아래는 실행문, cli에 입력하듯이 명령어를 작성하면 된다.
    run: |
          mysql --version
          sudo apt-get install -y mysql-client
          sudo service mysql start
          mysql -uroot -proot -e "CREATE DATABASE PLAM"
          mysql -uroot -proot -e "SHOW DATABASES"
      - name: Make .env file  # 새로운 액션의 이름
        uses: SpicyPizza/create-envfile@v1  # .env파일을 만들어주는 액션
        with:  # .env에 들어갈 값
     # 아래의 ${{secret.~~~}}는 레포지토리 secret에 저장해놓은 값을 사용
     # ${{job.services.mysql.ports['3306]}}은 위에서 설정한 값을 가져와서 3306과 매칭되는 포트값을 넣는 것
          envkey_PORT: 4000
          envkey_DOMAIN: plam.kr
          envkey_API_URL: http://127.0.0.1:4000
          envkey_STORAGE_URL: ${{secrets.STORAGE_URL}}
          envkey_DB_HOST: 127.0.0.1
          envkey_DB_PORT: ${{ job.services.mysql.ports['3306'] }}
          envkey_DB_USERNAME: root
          envkey_DB_PASSWORD: root
          envkey_DB_SCHEME_NAME: PLAM
          envkey_ADMIN_ID: ${{secrets.ADMIN_ID}}
          envkey_ADMIN_PASSWORD: ${{secrets.ADMIN_PASSWORD}}
          envkey_JWT_SECRET: ${{secrets.JWT_SECRET}}
          envkey_JWT_TOKEN_ADMIN: ${{secrets.JWT_TOKEN_ADMIN}}
          envkey_MAILGUN_API_KEY: ${{secrets.MAILGUN_API_KEY}}
          envkey_MAILGUN_SEND_NAME: ${{secrets.MAILGUN_SEND_NAME}}
          envkey_FCM_URL: ${{secrets.FCM_URL}}
          envkey_FCM_DYNAMIC_LINK_KEY: ${{secrets.FCM_DYNAMIC_LINK_KEY}}
          envkey_SMS_URL: ${{secrets.SMS_URL}}
          envkey_SMS_USER_ID: ${{secrets.SMS_USER_ID}}
          envkey_SMS_SECURE: ${{secrets.SMS_SECURE}}
          envkey_SMS_PHONE_1: ${{secrets.SMS_PHONE_1}}
          envkey_SMS_PHONE_2: ${{secrets.SMS_PHONE_2}}
          envKey_SMS_PHONE_3:
          envkey_AWS_S3_ACCESS_KEY_ID: ${{secrets.AWS_S3_ACCESS_KEY_ID}}
          envkey_AWS_S3_SECRET_ACCESS_KEY: ${{secrets.AWS_S3_SECRET_ACCESS_KEY}}
          envkey_AWS_S3_REGION: ap-northeast-2
      - name: install node modules && test # 테스트를 돌릴 액션
        run: |
          sudo service mysql restart
          npm install
          npm run reset
          npm test
        env: # 테스트시 필요한 환경설정
          CI: true # 지금 액션이 CI인 것을 명시
     # 아래는 데이터베이스 환경 설정
          MYSQL_DATABASE: PLAM
          DB_USERNAME: root
          DB_PASSWORD: root
          DB_PORT: ${{ job.services.mysql.ports[3306] }}
      - name: if fail  # 만약 실패했을 때의 설정
        uses: actions/github-script@v3  # 커스텀 스크립트를 사용하게 해주는 액션
        with:  # 커스텀 스크립트 작성란
          github-token: ${{github.token}}  # 깃허브 토큰으로 PR을 close시킴
          script: |
            const ref = "${{github.ref}}"
            const pull_number = Number(ref.split("/")[2])
            await github.pulls.createReview({
              ...context.repo,
              pull_number,
              body:"테스트코드를 다시 확인해주세요. ",
              event: "REQUEST_CHANGES"
            })
            await github.pulls.update({
              ...context.repo,
              pull_number,
              state: "closed"
            })
        if: failure()  # 실패했다는 이벤트가 발생하면 바로 위의 액션 실행
```

## 사용 후

---

### Test 중

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3c62365f-839b-4ff5-93aa-5391b25fb72a/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3c62365f-839b-4ff5-93aa-5391b25fb72a/Untitled.png)

### Test 실패

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3b3101f6-9c7a-4391-818a-9512040bd16a/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3b3101f6-9c7a-4391-818a-9512040bd16a/Untitled.png)

### Test 성공

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/75f293b4-9825-461e-8132-c5ccdcb95db5/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/75f293b4-9825-461e-8132-c5ccdcb95db5/Untitled.png)

## 생각해본 점

---

- 테스트 코드를 돌리는데 3분정도 걸리긴 하지만 3분날리고 나중에 유지보수하는데 시간 더 걸릴 때 보다 나은 것 같다.
- 하지만 로컬에서 돌리면 30초 정도에 테스트가 마무리되는데 의미 애매하다....🤷‍♂️
- 테스트 코드뿐만이 아니라 린트, 슬랙 알림 등등 활용할 점이 굉장히 많은 것 같다.

참고한 링크

- <https://github.com/features/actions>
- <https://zeddios.tistory.com/1047>
- <https://github.com/actions/github-script>
- <https://velog.io/@adam2/GitHub-Actions-%EB%A1%9C-%ED%92%80%EB%A6%AC%ED%80%98%EC%8A%A4%ED%8A%B8-test-%EA%B2%80%EC%A6%9D%ED%95%98%EA%B8%B0>
- <https://meaownworld.tistory.com/162>
- <https://meetup.toast.com/posts/222>
- <https://velog.io/@babycto/Github-Actions-eslint>
