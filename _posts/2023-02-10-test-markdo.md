---
layout: post
title: JS - 반복문 성능 차이에 대하여
subtitle: 반복문 성능 비교
tags: [JS]
comments: true
---

## for?? forEach??

NestJs로 설문 가져오기 API를 만들면서, for문을 무자비하게 썼었다.

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

위 코드가 처음 썼던 코든데, 이상하게도 너무 느렸다. (거의 3~5초 정도 걸렸다는...) 그냥 sql의 문제인가? 라고만 생각하고 넘어갔다.

근데 코드리뷰 진행중 `for` 대신 `forEach`를 쓰라는 리뷰를 받았다. `forEach`를 알긴해도 `for`이 더 편해서(모든 언어에 존재하니까!) 그것만 썼었는데, 생각해보니까 정확한 메커니즘 차이도 모르고 있었다. 이에 반복문의 메커니즘과 성능을 비교해보고자 한다.

##

<!--
This is a demo post to show you how to write blog posts with markdown. I strongly encourage you to [take 5 minutes to learn how to write in markdown](https://markdowntutorial.com/) - it'll teach you how to transform regular text into bold/italics/headings/tables/etc.

**Here is some bold text**

## Here is a secondary heading

Here's a useless table:

| Number | Next number | Previous number |
| :----- | :---------- | :-------------- |
| Five   | Six         | Four            |
| Ten    | Eleven      | Nine            |
| Seven  | Eight       | Six             |
| Two    | Three       | One             |

How about a yummy crepe?

![Crepe](https://s3-media3.fl.yelpcdn.com/bphoto/cQ1Yoa75m2yUFFbY2xwuqw/348s.jpg)

It can also be centered!

![Crepe](https://s3-media3.fl.yelpcdn.com/bphoto/cQ1Yoa75m2yUFFbY2xwuqw/348s.jpg){: .mx-auto.d-block :}

Here's a code chunk:

```
var foo = function(x) {
  return(x + 5);
}
foo(3)
```

And here is the same code with syntax highlighting:

```javascript
var foo = function (x) {
  return x + 5;
};
foo(3);
```

And here is the same code yet again but with line numbers:

{% highlight javascript linenos %}
var foo = function(x) {
return(x + 5);
}
foo(3)
{% endhighlight %}

## Boxes

You can add notification, warning and error boxes like this:

### Notification

{: .box-note}
**Note:** This is a notification box.

### Warning

{: .box-warning}
**Warning:** This is a warning box.

### Error

{: .box-error}
**Error:** This is an error box. -->
