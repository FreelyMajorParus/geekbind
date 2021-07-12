说明: 老师好，由于时间限制，本次作业提交仅涉及必做部分；</br>
代码库所在位置：https://github.com/FreelyMajorParus/JavaCourseCodes/commits/main/02nio/nio02 ， 仅涉及最近一次commit；

## 1. (必做）整合上次作业的 httpclient/okhttp；

说明: 继承上次作业中的HttpClient实现。
```java
private void fetchGet(final FullHttpRequest inbound, final ChannelHandlerContext ctx, final String url) {
        HttpGet httpGet = new HttpGet(url);
        CloseableHttpResponse response = null;
        InputStream content = null;
        try {
            response = httpClient.execute(httpGet);
            HttpEntity entity = response.getEntity();
            content = entity.getContent();
            int bytesAvailable = content.available();
            byte[] bytes = new byte[bytesAvailable];
            content.read(bytes, 0, bytesAvailable);
            DefaultFullHttpResponse responseTmp = new DefaultFullHttpResponse(HTTP_1_1, OK, Unpooled.wrappedBuffer(bytes));
            responseTmp.headers().set("Content-Type", "application/json");
            responseTmp.headers().set("Connection", "close");
            responseTmp.headers().set("Transfer-Encoding", "chunked");
//            responseTmp.headers().setInt("Content-Length", Integer.parseInt(response.getFirstHeader("Content-Length").getValue()));
            ctx.writeAndFlush(responseTmp);
            handleResponse(inbound, ctx, response);
        } catch (Exception ignore) {
            DefaultFullHttpResponse responseTmp = new DefaultFullHttpResponse(HTTP_1_1, OK, Unpooled.wrappedBuffer("inner error".getBytes()));
            responseTmp.headers().set("Content-Type", "application/json");
            responseTmp.headers().set("Connection", "close");
            responseTmp.headers().set("Transfer-Encoding", "chunked");
//            responseTmp.headers().setInt("Content-Length", Integer.parseInt(response.getFirstHeader("Content-Length").getValue()));
            ctx.writeAndFlush(responseTmp);
        } finally {
            try {
                if (content != null) {
                    content.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
            try {
                if (response != null) {
                    response.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```
在实现过程中，发现一个问题，代码刚拉下来之后按照自己的实现，没有添加`responseTmp.headers().setInt("Content-Length", Integer.parseInt(response.getFirstHeader("Content-Length").getValue()));`，服务启动之后，发现浏览器能够拿到数据，但是一直在转圈，排查了很久才发现这个Content-Length的问题，在HTTP 1.1，如果当前连接属于"Keep-alive"的长连接，则要么提供Content-length,要么提供Transfer-Encoding参数，这样客户端才能知道要取多少数据。(ps. 如果是Transfer-Encoding，则代表数据采用分块编码方式发送，该连接发送的最后一个分块的长度是0)。为了实现如上逻辑，故将Content-length注释，使用Transfer-Encoding代替，最终请求正常。


##  3. (必做）实现过滤器。
说明: 该过滤器非自己从0开始实现，由于拉代码发现老师的代码已经在上面，后来看了下老师的代码，inbound的逻辑与具体的Filter耦合在一起，后期要加其他filter，改造都比较大，参照java web对过滤器的设计，如果能够动态加入多个filter，对后期的扩展也比较友好。从我的观点出发，将现有的过滤器进行改造如下, 如有过度设计或存在设计的坏味道，麻烦老师指出。

3.1 修改原先的顶层抽象RequestFilter。 添加sort()以及code(),sort主要用于多个过滤器之间的排序，而code主要用于去重。
```java
public interface HttpRequestFilter {
    
    void filter(FullHttpRequest fullRequest, ChannelHandlerContext ctx);

    int sort();

    String code();
}
```
3.2 引入适配器层-HttpRequestFilterAdapter, 提供默认的sort()以及code()实现，面向提供普通的Filter的方式。
```java
public abstract class HttpRequestFilterAdapter implements HttpRequestFilter,Comparable<HttpRequestFilterAdapter> {
    private static final String DEFAULT_CODE = "default";

    @Override
    public int sort() {
        return 0;
    }

    @Override
    public String code() {
        return DEFAULT_CODE;
    }

    @Override
    public int compareTo(HttpRequestFilterAdapter o) {
        return Integer.compare(this.sort(),o.sort());
    }
}
```
3.3 让原先的HeaderHttpRequestFilter继承自适配类，仅提供基础的过滤器功能。
```java
public class HeaderHttpRequestFilter extends HttpRequestFilterAdapter {
    @Override
    public void filter(FullHttpRequest fullRequest, ChannelHandlerContext ctx) {
        fullRequest.headers().set("mao", "soul");
    }
}
```
3.4 当我们有其他的过滤器要加入时，如下实现:
```java
public class HeaderHttpRequestExtraFilter extends HttpRequestFilterAdapter{

    @Override
    public void filter(FullHttpRequest fullRequest, ChannelHandlerContext ctx) {
        fullRequest.headers().set("name", "major.parus");
    }

    @Override
    public int sort() {
        return -1;
    }

    @Override
    public String code() {
        return "extra";
    }
}
```
3.5 引入过滤器容器，统一管理; 主要完成打包一系列过滤器的功能，并且实现HttpRequestFilterAdapter接口，作为容器类的过滤器也要提供过滤的功能。
```java
public class HeaderHttpRequestFilterContainer extends HttpRequestFilterAdapter{

    public LinkedHashSet<HttpRequestFilter> container = new LinkedHashSet<>();

    public HeaderHttpRequestFilterContainer packaging(HttpRequestFilter filter) {
        container.add(filter);
        return this;
    }

    @Override
    public void filter(FullHttpRequest fullRequest, ChannelHandlerContext ctx) {
        container.stream().sorted().collect(Collectors.toList()).forEach((filter) -> {
            filter.filter(fullRequest, ctx);
        });
    }
}
```
3.6 替换原有的过滤器耦合部分，引入容器类，包装所需要的过滤器，最终以过滤器的身份被调用.
```java
public class HttpInboundHandler extends ChannelInboundHandlerAdapter {

    private final List<String> proxyServer;
    private HttpOutboundHandler handler;
    private HeaderHttpRequestFilterContainer filters = new HeaderHttpRequestFilterContainer();
    public HttpInboundHandler(List<String> proxyServer) {
        this.proxyServer = proxyServer;
        this.handler = new HttpOutboundHandler(this.proxyServer);
        filters.packaging(new HeaderHttpRequestFilter()).packaging(new HeaderHttpRequestExtraFilter());
    }
    ...
}
```
总结: 主要就是要是将涉及到的过滤器放进一个容器中，而其他的Inbound组件仅需要依赖容器，可以根据需要将不同过滤器包装到容器中。
