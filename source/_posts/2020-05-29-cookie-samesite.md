---
title: Cookie SameSite 食用指南
date: 2020-05-29 16:29:30
tags: [WEB]
abstract: Cookie's attribute SameSite will change the Web Security World
header_image: /intro/post-bg-huiye.png
top:
---

# The foregoing.
随着Web安全的发展需要，浏览器端已经开始针对Cookie进一步防御，而SameSite属性正是这场防御进化的焦点。Samesite属性于2016年由IETF正式发布，现今出于网站稳定性的考量（理同废除Flash），大部分浏览器采用支持但默认不开启状态（SameSite=None），但该属性一定会随时间缓步推进普及，并最终适用于主流浏览器内。
故，遵从“先知为敬”的战略，本文就来初步探讨一下Samesite属性基础以及其对客户端安全性的影响。

<br/>

# PART1: SameSite explained
## What is SameSite Cookie
`SameSite`作为一种属性存在于Cookie中，类似于`Domain`、`HttpOnly`等属性。**如同其名，Samesite主要用于说明该Cookie是否仅被同站请求享有，来控制Cookie何时该被跨站请求携带**。我们先来看看它的三个值：
1. None: 该Cookie不启用`SameSite`，即任何类型的请求都会携带该Cookie。截止文章书写日期，所有支持SameSite的浏览器均对没设置该属性的Cookie采用`SameSite=None`。未来将对设置为None的Cookie强制要求设置Secure为True。
2. Lax: 采用宽松的策略，只要同站请求或者是部分跨站请求即会携带该Cookie。未来浏览器SameSite将默认设置为该值。
3. Strict: 采用严格的策略，仅在同站请求中携带该Cookie。

## LAX?
看到这里，读者的第一个疑问应该是，宽松策略到底宽松到什么样的程度？关于这个问题，IETF在[文档](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-same-site-00)中写道：
> Safe" HTTP methods include "GET", "HEAD", "OPTIONS", and "TRACE", as defined in Section 4.2.1 of [RFC7231].
> ......
> Lax enforcement provides reasonable defense in depth against CSRF attacks that rely on unsafe HTTP methods (like "POST"), but do not offer a robust defense against CSRF as a general category of attack:
> 1.  Attackers can still pop up new windows or trigger top-level navigations in order to create a "same-site" request (as described in section 2.1), which is only a speedbump along the road to exploitation.
> 2.  Features like "<link rel='prerender'>" [prerendering] can be exploited to create "same-site" requests without the risk of user detection.

简单总结一下，只要跨站请求属于"GET"、"HEAD"、"OPTIONS"或"TRACE"中一种，且通过顶级导航栏发送请求，那么在`SameSite=LAX`下，也会携带该Cookie的。
从实现上我们可以想到，设计师把它作为默认值是为了防止存在跨站跳转页面的场景中，Cookie不被携带而导致功能失效。当然，Lax下可见安全防护性也偏低。

## Same? Site?
SameSite对同站请求的定义是十分重要的，具体可以参考[IETF关于same-site和cross-site的定义](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-same-site-00#section-2.1)。需要注意的是，有些web手会在这里把samesite和同域策略画等号，其实同站会比同域的要求低很多，不需要同时符合协议、域名、端口号。
简单来说，除了`Refer`为空等请求属于同站请求外，我们可以通过辨别Refer和Host的Second-Level Domain（二级域名，sld）是否一致判断是否为同站请求，而二级域名是由[public suffix](https://publicsuffix.org/list/public_suffix_list.dat)（Mozilla维护的顶级域名）和其后一级域名构成。
理论繁复，我们可以通过工具来帮助我们判断同/跨站请求，python模块publicsuffix2可以更清晰地提取某域名的二级域名：
```
$ sudo pip3 install publicsuffix2
$ python3
>>> from publicsuffix2 import get_sld
>>> get_sld('www.foo.com')
'foo.com'
>>> get_sld('www2.foo.com')
'foo.com'
>>> get_sld('foo.github.io')
'foo.github.io'
>>> get_sld('jack.github.io')
'jack.github.io'
```
从提取的二级域名可以判断，域名为`www.foo.com`和`ww2.foo.com`的两个请求属于同站请求，域名为`foo.github.io`和`jack.github.io'`的两个请求为跨站请求。

## Browser support
对于SameSite属性的实现，大部分主流浏览器已完成，等待的是何时将[未来改进措施](https://tools.ietf.org/html/draft-west-cookie-incrementalism-00#section-3)纳入实施（默认值设为Lax，值为None时需将Secure置为Ture）
* SameSite属性实现情况：
![samesite-support](./samesite-support.png)
* SameSite默认值设为Lax及上下文需求Secure：
![samesite-incrementalism](./samesite-incrementalism.png)

需要说明的是，后一项仅是实现该功能，即可以开启，但不默认启用。离SameSite的相关提案正式普遍实施还为时尚早。读者阅读时的浏览器支持，请访问Can I Use自行查看。

<br/>

# PART2: SameSite means what?
SameSite文档和网络上很多文章都提到，SameSite是为深度抵御CSRF攻击而推出的。那么它实际上到底能缓解什么类型的攻击？这需要先从本质上理解SameSite的意义。根据PART1提供的原理，**SameSite默认启用Lax状态下，其实际上是在阻止外站访问本站时，本站Cookie的携带**（除了顶级导航栏跳转），而这种跨站请求时常发生在身份验证/伪造上。深刻理解上面这段话，我们就能分析什么Web攻击将会受到它的影响。

## Effects on CSRF
CSRF中文翻译就是跨站请求伪造，字字被Part2开头点到，肯定受到了很大的限制。从CSRF本质上来看，它利用了浏览器跨站访问时，会自动携带目标站点Cookie的特性，而SameSite正是说明了什么Cookie该在跨站时携带。从CSRF攻击场景上来看，非结合XSS的CSRF，攻击者都需要在其它站点搭建恶意页面引导受害者访问，恶意站点实现的跨站请求需要根据实际攻击的站点功能所决定，而Lax值下，仅允许顶部导航的跨站携带该Cookie，POST等非安全HTTP方法下，CSRF将会受到限制。（POST又是CSRF的常用攻击手段）
然而在SameSite应用初期，浏览器为了保证大部分站点的功能正常运行，即使手动开启默认SameSite=Lax，也会受到浏览器缓和措施的影响，而使CSRF攻击面没那么狭窄。例如SameSite先行者Chrome会将未设置SameSite的Cookie两分钟内依旧可以被POST携带( https://www.chromestatus.com/feature/5088147346030592 )，利用这项机制，两分钟内发动攻击，或利用应用程序相关功能刷新Cookie依旧可以让CSRF所向披靡。相关研究已经有部分研究员发布了成果和想法，如：https://medium.com/@renwa/bypass-samesite-cookies-default-to-lax-and-get-csrf-343ba09b9f2b 。当然，对于令牌存储到localStorage的情况，CSRF将不会受到影响。

## Effects on Hijacking
对于劫持，特别是点击劫持，如果从整个攻击发生过程上来看，你会发现两种漏洞很相似。点击劫持在用户浏览器使用iframe隐秘加载被攻击站时，也是种跨站请求，且不属于顶级导航跳转，不设置SameSite的Cookie是不会携带的，从而缺乏用户认证所需令牌，无法完成相应的攻击。对于令牌存储到localStorage同CSRF。

## Effects on Other Web Attacks
我们以XSSI为例来描述一下其余可能受到影响的Web漏洞。从攻击场景上来看，XSSI需要嵌入一个带身份验证的资源进script标签以绕过SOP限制，从而窃取用户可能的隐私信息。然而，若SameSite生效，包含在Cookie中的用户令牌将不会插入到请求当中，正常情况下会让该攻击不能成功。从上述三个漏洞影响分析中我们自然可以发现，只要攻击涉及到利用用户身份从攻击站访问受害站时，SameSite会限制相关攻击的中间流程，以此推倒，很多隐私泄漏型等攻击将会或多或少收到影响，读者可以举一反三，将本文的经验用于类比分析。

<br/>

# Afterword.
SameSite作为一种Cookie属性，弥补了跨站/跨域问题上，浏览器特性导致的缺口。分析下来，其可以抵抗的范围广，影响力大，是一种足以推动Web安全向前进步的补足。相信在不久的将来，其实际应用之际，会给Web增添新光彩。

<br/>

# Reference
* IETF SameSite文档：https://tools.ietf.org/html/draft-ietf-httpbis-cookie-same-site-00
* IETF SameSite改进措施：https://tools.ietf.org/html/draft-west-cookie-incrementalism-00#section-3
* 长亭推文 - CSRF 漏洞的末日？https://mp.weixin.qq.com/s/YqSxIvbgq1DkAlUL5rBtqA
* Web.dev博客普及SameSite: https://web.dev/samesite-cookies-explained/
* Mozilla公共后缀列表：https://publicsuffix.org/
* Chrome相关说明：https://www.chromestatus.com/feature/5088147346030592  &  https://www.chromium.org/updates/same-site/faq
* Twitter@RenwaX23关于SameSite的绕过: https://medium.com/@renwa/bypass-samesite-cookies-default-to-lax-and-get-csrf-343ba09b9f2b
* Twitter@filedescriptor,@ngalongc,@EdOverflow关于SameSite影响面分析： https://blog.reconless.com/samesite-by-default/