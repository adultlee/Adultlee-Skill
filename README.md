# Adultlee-Skill

## TryMake 
> TryMake는 (주) 마로마브에서 인턴(파트타임)으로 진행한 Next.js 기반 웹뷰 프로젝트 입니다. 

### Trouble Shooting - 초기 렌더링 최적화 문제

TryMake 내부의 Editor는 Scratch 라이브러리 기반으로 동작하여, 초기 렌더링 속도가 지나치게 느린 문제가 존재했습니다. 이는 유저의 이탈률을 높이는 지표로서 나타났습니다.
<div>
<img width="484" alt="image" src="https://github.com/adultlee/Adultlee-Skill/assets/77886826/2c1d574e-c4c6-4908-a3a7-9a490a4eaeab">
<img width="446" alt="image" src="https://github.com/adultlee/Adultlee-Skill/assets/77886826/eabce43c-bf03-410f-a197-ac37a36adc59">
</div>

기존의 scratch와의 전송된 데이터량의 차이가 약 7배 정도 발생했었으며, content-length 도 그에 준하게 약 5배 가량 차이나는 것을 확인할 수 있었습니다.

같은 파일인데도 content-length에 큰 차이가 있는 이유는 바로 compression 때문입니다.

CloudFront로 배포하는 경우 CloudFront 정책 상 자동 압축하여 컨텐츠를 제공하게 됩니다.
하지만 저희는 압축되지 않고 있었는데... 

그 이유는 CloudFront의 압축 제한이** 10,000,000 byte**이기 떄문입니다.
하지만 저희가 배포했던 최초 파일의 크기는 2천만 바이트... 당연히 CloudFront에서 자동으로 압축이 진행되지 않았던 것입니다.

따라서 최초로 빌드해서 배포할 때 압축해서 배포해야합니다.

```js
new HtmlWebpackPlugin({
  chunks: ['lib.min', 'gui'],
  template: 'src/index.ejs',
  title: 'MAKE : Code editor',
  sentryConfig: 시크릿키
    ? '"' + 시크릿키 + '"'
    : null,
  jsExtension: '.gz', // 여기가 중요~~~
}),
new CompressionPlugin({
  include: ['lib.min.js', 'chunks/gui.js'],
}),
new HtmlWebpackChangeAssetsExtensionPlugin(), // 실제 사이즈가 굉장히 큰 파일들만 압축하기 위해서 압축할 파일을 직접설정한 후 사용
// CompressionPlugin과 HtmlWebpackPlugin을 같이 사용하는 경우에 index.html에 압축파일이 제대로 포함이 안되기 때문에 해당 문제를 해결하기 위해서 추가한 플러그인
```

이 상태로 번들을 하게 되면, lib.min.js.gz, chunks/gui.js.gz가 생성되며

아래와 같이 index.html이 생성된다.

```html
// 번들 결과
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, initial-scale=1, maximum-scale=1.0, user-scalable=no"
    />
    <meta name="google" value="notranslate" />
    <link rel="shortcut icon" href="static/favicon.ico" />
    <title>MAKE : Code editor</title>
    
  </head>
  <body><script type="text/javascript" src="lib.min.js.gz"></script><script type="text/javascript" src="chunks/gui.js.gz"></script></body>
</html>
```

하지만 여기서 끝이 아니라 gzip파일의 경우 S3에서 제공해 줄 때, Content-encoding: gzip 을 넣어주어야 합니다. (Editor는 SPA로 구성되어 S3에 배포합니다.)

이는 콘솔에서 수동으로도 작업이 가능하지만 그렇게 할 경우 CD가 제대로 되지 않기 떄문입니다. 배포할 때마다 콘솔에 들어가서 수정해주어야 하기 때문에 CD 스크립트에서 수정해 주었습니다.

```
- name: Upload to S3 exclude gz files
  run: ------------------------------------------------
  --exclude "*.gz" --recursive

- name: Upload to S3 gz files
  run: ------------------------------------------------
	--exclude="*" --include="*.js.gz" --include="*.js.map.gz" 
  --content-encoding gzip --content-type="application/javascript" 
  --cache-control "max-age=31536000" 
  --metadata-directive REPLACE --recursive
```

다음과 같이 gz파일과 그렇지 않은 파일을 분리해서 업로드해줍니다.

처음엔 gz파일을 제외하고 업로드 한 뒤 모든 파일을 제외하고 gz파일만 포함해서 업로드하는 과정을 거칩니다. 그 다음 content-encodig을 gzip으로 명시해주면 됩니다.

### 결과
<img width="725" alt="image" src="https://github.com/adultlee/Adultlee-Skill/assets/77886826/c218914d-bad4-4084-a240-8e1141058b64">
> 모든 컨텐츠를 로딩하는데 까지 5.9s -> 2.5s 

![_-ezgif com-speed](https://github.com/adultlee/Adultlee-Skill/assets/77886826/c8cd3bdf-1761-40d8-8324-cbb13830d15f)

### Trouble Shooting - 크로스 브라우징 대응
TryMake 서비스는 일반적으로 앱에서 사용되는 세로방향 서비스가 아닌 가로방향 서비스입니다.

가로 방향에서의 하단 keyboard 이벤트에 대응하며 iOS 15 버전이 업데이트 됨에 따라 댓글을 입력하는 하단 폼이 하단에 위치하는 점은 UX에 크나큰 문제가 되었습니다.

<img width="225" alt="image" src="https://github.com/adultlee/Adultlee-Skill/assets/77886826/3ee97b40-7d45-407c-a08b-fa1dc10d9c67">

> iOS가 업데이트 되며 하단에 주소창이 생기는 등, 키보드 입력 이벤트가 발생함에 따라 입력창이 가려지는 문제가 발생

-> visualViewport를 사용하여 문제 해결

```js
  useEffect(() => {
    visualViewport.addEventListener("resize", () => {
      const deviceHeight =
        screen.width < screen.height ? screen.width : screen.height;

      if (visualViewport.height < deviceHeight - 150) {
        setCommentBarOpened(true);
      } else {
        setCommentBarOpened(false);
      }
    });
  }, []);
```
### 결과

![_-ezgif com-speed (1)](https://github.com/adultlee/Adultlee-Skill/assets/77886826/0cb0bdd9-727b-407e-824d-1bf11efa607f)

![KakaoTalk_Video_2022-11-09-21-36-10-ezgif com-video-to-gif-converter](https://github.com/adultlee/Adultlee-Skill/assets/77886826/e4705287-c20d-4ad2-8d3b-e290481595f1)



