---
layout: post
title: GzipFilter
date: 2024-11-11 13:20:23 +0900
category: was
---

## GzipFilter?
일부 WAS는 GzipFilter라는 Filter를 지원한다.

`GzipFilter`의 목적은 무엇일까?

일반적으로 GzipFilter는 Server의 응답을 Gzip으로 압축하여 Client로 전달하는 역할을 한다.<br/> 데이터 형태나 압축 방식에 따라 압축율의 차이는 있겠지만 전송되는 데이터 양을 줄여서 적절한 환경에서는 성능 향상을 기대할 수 있다.

## GzipFilter 동작 조건
당연한 이야기지만 `GzipFilter`가 온전하게 사용되려면 압축된 응답을 수신하는 Client가 압축을 해제할 수 있어야 한다. 따라서 Client는 Server에게 요청을 전송할 때 자기가 압축을 해제할 수 있는 Client라는 정보를 함께 전달한다.

방법은 아래와 같이 요청 헤더에 gzip값을 추가한다.

<span style="color: red;">Accept-Encoding: gzip</span>

서버는 위 요청 헤더를 확인하고 응답을 압축하면 반대로 Client에게 이것이 압축된 응답이라는 것을 알린다. 방법은 요청과 비슷하게 헤더의 값을 셋팅하면 된다.

<span style="color: red;">Content-Encoding: gzip</span>

## 동작원리
사실 GzipFilter의 동작원리는 어려운 개념이 아닌 것 같다.<br/> 응답 전문을 Gzip으로 압축하여 전달해주면 되기 때문이다.


## 구현
그럼 Application단에서도 손쉽게 구현이 가능하지 않을까? 일단 시도해보도록 하자

### 1. Filter클래스 생성
먼저 filter 클래스 를 implements한 Gzip클래스를 생성하고 doFilter 메서드를 구현해보자 gzip이 있을 경우 응답 헤더에도 gzip을 셋팅한다.

```java
      @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException 
    {
        HttpServletRequest httpRequest = (HttpServletRequest)request;
        HttpServletResponse httpResponse = (HttpServletResponse)response;

        String acceptEncoding = httpRequest.getHeader("Accept-Encoding");

        if(acceptEncoding != null && acceptEncoding.contains("gzip"))
        {
            httpResponse.addHeader("Content-Encoding", "gzip");
            // gzip compress
        }
        else
            chain.doFilter(request, response);
    }
```

### 2. Gzip Response Stream
관건은 Response Stream에 정상적으로 압축된 데이터를 기록하는 것이다. <br/>
비즈니스 로직에서 OutputStream을 가져갈 때 GzipOutputStream을 가져가게 하기 위해 ResponseWrapper를 구현 해보자.

```java
    private class GzipHttpServletResponseWrapper extends HttpServletResponseWrapper
    {
        private GzipServletOutputStream gzipOutputStream = null;
        private PrintWriter printWriter = null;

        public GzipHttpServletResponseWrapper(HttpServletResponse response) {
            super(response);
        }

        @Override
        public ServletOutputStream getOutputStream() throws IOException
        {
            if(this.gzipOutputStream == null)
            {
                this.gzipOutputStream = new GzipServletOutputStream(getResponse().getOutputStream());
            }
            return this.gzipOutputStream;
        }

        @Override
        public PrintWriter getWriter() throws IOException
        {
            if(this.printWriter == null)
            {
                this.gzipOutputStream = new GzipServletOutputStream(getResponse().getOutputStream());
                this.printWriter = new PrintWriter(new OutputStreamWriter(this.gzipOutputStream, getResponse().getCharacterEncoding()));
            }
            return this.printWriter;
        }
    }

    private class GzipServletOutputStream extends ServletOutputStream
    {
        private GZIPOutputStream gzipOutputStream = null;

        public GzipServletOutputStream(OutputStream out) throws IOException
        {
            super();

            this.gzipOutputStream = new GZIPOutputStream(out);
        }

        @Override
        public void close() throws IOException
        {
            this.gzipOutputStream.close();
        }

        @Override
        public void flush() throws IOException
        {
            this.gzipOutputStream.flush();
        }

        @Override
        public void write(byte b[]) throws IOException
        {
            this.gzipOutputStream.write(b);
        }

        @Override
        public void write(byte b[], int off, int len) throws IOException
        {
            this.gzipOutputStream.write(b, off, len);
        }

        @Override
        public void write(int b) throws IOException
        {
            this.gzipOutputStream.write(b);
        }
    }
```

## 결과
정상동작이 가능한지는 위의 동작원리에서 설명한 것과 같이 응답에 gzip헤더가 내려오고

응답 사이즈가 줄어드는 것으로 확인 할 수 있다.

물론 동작도 잘 되는 것처럼 보이지만 이 코드는 제약이 존재한다. 

그 제약은 다음 글에서 알아보도록 하자.