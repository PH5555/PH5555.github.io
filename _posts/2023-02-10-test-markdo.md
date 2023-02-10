---
layout: post
title: JS - 배열의 비동기 처리
tags: [JS]
comments: true
---

## forEach는 비동기함수를 기다리지 않는다

아래 코드는 각 설문 문항에 맞는 답변을 가져오는 코드다.

```javascript
for (const question of survey["surveyQuestions"]) {
  answers.push(
    await this.findOne({
      relations: ["question", "answerType", "singleChoiceAnswers"],
      where: {
        question: {
          questionId: question.questionId,
        },
      },
    })
  );
}
```

문항 배열을 for로 순환하며 답변을 가져오는 형태이다. 하지만 이상하게도 너무너무너무 느렸다... 그리고, 코드리뷰 진행중 `for` 대신 `forEach`를 쓰라는 리뷰를 받았는데

```javascript
survey["surveyQuestions"].forEach(async (question) => {
  const { questionId } = question;
  const questionRelation = ["question", "answerType", "singleChoiceAnswers"];
  const questionQuery = {
    question: { questionId },
  };
  const answer = await this.findOne({
    relations: questionRelation,
    where: questionQuery,
  });
  answers.push(answer);
});
```

로직을 수정해보니까 for로 했을때와 달리 값이 정상적으로 실행되지 않았다. 찾아보니 for과 달리 forEach는 비동기 함수를 기다려주지 않는다고 한다. 즉, 내부 코드에 await가 있던 없던 신경 안쓰고 넘어가 버린다.

## Promise.all 을 이용한 병렬 처리

forEach가 비동기 함수를 기다려주지 않는 문제는 처음 코드와 같이 for 문으로 해결할 수 있다. 하지만 너무너무너무 느리다...! 그래서 map과 Promise.all을 이용해 병렬처리를 한다.

```javascript
const answers = [];

const promises = survey["surveyQuestions"].map(async (question) => {
  const { questionId } = question;
  const questionRelation = ["question", "answerType", "singleChoiceAnswers"];
  const questionQuery = {
    question: { questionId },
  };
  const answer = await this.findOne({
    relations: questionRelation,
    where: questionQuery,
  });

  answers.push(answer);
});

await Promise.all(promises);
```

Promise.all은 비동기 함수들을 순차적으로 처리하지 않고, 한번에 병렬처리하기 때문에 for 보다 속도가 월등히 빨라진다.

## 그렇다면 ForEach도 병렬처리인가?

forEach를 썼을 때 비동기함수가 한번에 처리되니까 병렬처리일까? 라는 생각을 했지만 공식 문서를 찾아보니 `순차적`으로 처리한다고 나와있었다.

> forEach()는 주어진 callback을 배열에 있는 각 요소에 대해 오름차순으로 한 번씩 실행합니다. 삭제했거나 초기화하지 않은 인덱스 속성에 대해서는 실행하지 않습니다. (예: 희소 배열)

이 글을 쓰다보니 병렬처리에 대한것이 너무 헷갈린다... 다음 포스팅에서 자세히 알아보도록 하자
