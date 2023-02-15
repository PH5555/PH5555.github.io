---
layout: post
title: nestJs - 트랜잭션 사용하기
tags: [nestJs, backend]
comments: true
---

## 트랜잭션이 필요한 이유?

설문조사 결과를 저장하는 API 구현 중 여러 테이블에 데이터를 INSERT 해야했다.

여러 테이블에 데이터를 차례대로 집어넣는 중 한 테이블에서 에러가 발생하면, 이전에 집어넣었던 데이터는 어떻게 되어야할까?

설문조사 결과를 저장하는 일련의 과정을 트랜잭션이라고 할 수 있다. 트랜잭션의 가장 중요한 특징 중 하나는 작업이 **모두 완료되거나** **모두 완료되지 않아야 한다.** 즉 한 테이블에서 에러가 발생하면, 롤백 시켜서 이전 테이블 상태로 돌아가야한다.

## TypeOrm 에서의 트랜잭션

typeorm 에서 트랜잭션을 작성하는 방법에는 여러가지가 있다. 여기서 내가 선택한 방법은 queryRunner를 이용한 방법이다.

```typescript

const connection = getConnection();
const queryRunner = connection.createQueryRunner();

await queryRunner.connect();
await queryRunner.startTransaction();

try {
  ...
  await queryRunner.commitTransaction();
} catch (err) {
  await queryRunner.rollbackTransaction();
} finally {
  await queryRunner.release();
}

```

queryRunner를 생성하고 `startTransaction`으로 트랜잭션을 실행할 수 있다.

트랜잭션에는 커밋, 롤백, 릴리즈가 있다. 커밋을 하면 데이터베이스 수정사항을 적용할 수 있고, 롤백을 하면 데이터베이스 수정사항을 적용하지 않고 이전으로 돌아갈 수 있다. 그리고 릴리즈를 하면 트랜잭션을 해제할 수 있다. 각각 `commitTransaction`, `rollbackTransaction`, `release` 메서드를 이용해서 실행할 수 있다.

## 근데 왜 안돼? (auto commit)

위 방법으로 내 로직을 트랜잭션으로 처리했다. 그리고 테스트를 위해 고의적으로 에러를 발생시켜보았다. 에러가 발생하긴하나 롤백이 되지 않고, 에러가 나지 않은 부분만 커밋이 되어 버렸다.

무슨 영문인지 좀 헤매다가 nestJs에 쿼리로그를 볼 수 있는 기능이 있어 한번 해보았다. 문제는 repository를 이용해 데이터베이스에 접근을 할 때마다 커밋이 자동으로 되어버리는 것이었다. `autoCommit` 기능을 false로 만들고 `Transactional` 데코레이터를 붙여서 손쉽게 처리할 수 있지만, 위 방법처럼 명시적으로 트랜잭션을 코드상에서 처리하는 것이 더 좋은 방법이라하여 딴 방법을 찾아보았다.

https://stackoverflow.com/questions/69221128/how-to-use-transactions-with-nestjs-repositories

```typescript
const connection = getConnection();
const queryRunner = connection.createQueryRunner();
const queryManager = queryRunner.manager;

await queryRunner.connect();
await queryRunner.startTransaction();

try {
  const surveyResult = await queryManager
    .getCustomRepository(SurveyResultRepository)
    .createSurveyResult();
  surveyResultId = surveyResult.resultId;

  const promises = contents.map(async (content) => {
    const questionId = content.question_id;
    const surveyQuestion = await queryManager
      .getCustomRepository(SurveyQuestionRepository)
      .findOne(questionId);

    const surveyAnswerResult = await queryManager
      .getCustomRepository(SurveyAnswerResultRepository)
      .createSurveyAnswerResult(surveyResult, surveyQuestion);

    const promises = content.answers.map(async (answer) => {
      // if(!isEmpty(answer.answer_content)) // 주관식 처리
      const surveyAnswer = await queryManager
        .getCustomRepository(SingleChoiceAnswerRepository)
        .findOne(answer.id);

      await queryManager
        .getCustomRepository(SurveySingleChoiceAnswerResultRepository)
        .createSingleChoiceAnswerResult(surveyAnswerResult, surveyAnswer);
    });

    await Promise.all(promises);
  });

  await Promise.all(promises);
  await queryRunner.commitTransaction();
} catch (err) {
  await queryRunner.rollbackTransaction();
  if (err["code"] === "23502" && err["column"] === "question_id")
    throw new BadRequestException("잘못된 question_id 입니다");
  if (err["code"] === "23502" && err["column"] === "single_choice_answer_id")
    throw new BadRequestException("잘못된 answer_id 입니다");
} finally {
  await queryRunner.release();
}
```

위 해결법에 따라 주입된 레파지토리를 사용하지 않고 queryRunner로부터 얻은 manager로 데이터를 조작하니 정상 작동을 하였다!

## 한가지 의문점

데이터가 롤백될 때 auto increment에 사용되는 sequence는 초기화되지 않았다. 사실 크게 문제될거 같지는 않지만 내가 잘못 구현을 한것인지, 초기화가 되지 않아도 상관이 없는지 찾아봐야할 거 같다.
