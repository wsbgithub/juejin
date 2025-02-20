﻿在现代 Web 开发中，动画已经成为吸引用户并增强用户体验的不可或缺的一部分。动画可以让 Web 应用或页面内容更具吸引力，提供信息反馈，以及增加互动性。而在创建令人印象深刻的动画时，CSS 动画合成（`animation-composition`）特性给 Web 开发者开发动画效果带来更强大的力量。

  


CSS 动画合成（`animation-composition`）是 CSS 中的一个相对较新的特性，它使 Web 开发者能够更灵活地组合、控制和定时多个 CSS 动画（`animation`），以创建更为复杂的动画效果。这一功能不仅简化了动画的创建过程，还允许 Web 开发者在不依赖 JavaScript 的情况下实现复杂的动画场景。

  


这节课我将带领你深入了解 CSS 动画合成的核心概念和用法，以及如何使用它来创建令人印象深刻的动画。无论你是初学者还是有经验的开发者，都能从这节课中获得有关于 CSS 动画合成的宝贵知识。

  


让我们一起探索 CSS 动画合成的奇妙世界，为你的 Web 项目添加生动和吸引力的动画效果吧！

  


## 多动画存在的问题

  


使用 CSS 的 `animation` 制作 Web 动画的时候，或许会在同一个元素上使用两个或两个以上的 CSS 动画，比如下面这个动画效果：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fad0ef9f84de402988983ed09ed9e076~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=994&h=648&s=402952&e=gif&f=129&b=f3f3f3)

  


这是一个很简单地 CSS 动画。如果将其拆分的话，动画元素（圆点）由两个简单的动画组成：

  


-   第一个是圆点沿着 `x` 轴移动一定的距离
-   第二个是圆点沿着 `y` 轴移动一定的距离

  


如果使用 `@keyframes` 来描述这两个动画的话，它可能像下面这样：

  


```CSS
@keyframes xAxis {
    50% {
        animation-timing-function: ease-in;
        translate:450px 0;
    }
}

@keyframes yAxis {
    50% {
        animation-timing-function: ease-out;
        translate:0 -450px;
    }
}
```

  


并且将这两个动画都运用到动画元素 `.dot` 上：

  


```CSS
.dot {
    animation: 
        xAxis 2.5s infinite cubic-bezier(0.02, 0.01, 0.21, 1),
        yAxis 2.5s infinite cubic-bezier(0.3, 0.27, 0.07, 1.64);
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ed10ed0af2d4a3ca6f381fd29f00382~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1180&h=594&s=804759&e=gif&f=88&b=eeeeee)

  


> Demo 地址：https://codepen.io/airen/full/PoXRPod

  


并不能发现，和我们期望的动画效果相差甚远：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8389440ef15e48fb9122e770f38d7d56~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1130&h=596&s=968894&e=gif&f=154&b=e8e8e8)

  


> Demo 地址：https://codepen.io/airen/full/RwEMWoB

  


你可能会好奇，这到底是为什么呢？

  


这里简单的阐述一下。在这个示例中，`xAxis` 和 `yAxis` 都是使用 CSS 的 `translate` 属性对动画元素进行位移，不同的是，`xAxis` 使动画元素沿着 `x` 轴水平向右移动（即 `translate: 450px 0`），而 `yAxis` 使动画元素沿着 `y` 轴垂直向上移动（即 `translate: 0 -450px`）。可是，在同一个元素，同一时间只能设置一个变换值（即，只有一个 `translate` 可用于目标元素上），因此只有一个 `translate` （位移）动画会生效。因此，在动画列表中，`yAxis` 动画会覆盖 `xAxis` 动画。

  


事实呢？在同一个元素上使用多个 CSS 动画除了性能问题以及增加维护和调试困难之外还会增加冲突，甚至让动画不可控，造成混乱。

  


-   **冲突**：不同的动画可能会同时修改相同的 CSS 属性，导致冲突和不一致。例如，一个动画试图将元素向左移动，而另一个动画同时试图将元素向右移动
-   **不可控**：多个动画可能会导致元素的状态变得复杂，难以预测和控制。这可能会导致不可预测的动画效果。例如，一个动画试图将元素向右移动，而另一个动画同时试图将元素向上移动
-   **混乱**：多个动画可能会导致动画效果重叠，使元素的行为变得混乱。例如，一个动画正在逐渐淡出元素，而另一个动画同时试图将元素放大

  


为了避免这些问题，开发者通常需要仔细规划和组织他们的 CSS 动画，确保它们不会相互干扰，并尽量减少多个动画同时作用于同一个元素的情况。例如，我们一般采用分层来修复上面示例：

  


```HTML
<div class="dot xAxis">
    <span class="yAxis"></span>
</div>
```

  


-   父元素运用 `xAxis` 动画，使其沿着 `x` 轴向右移动
-   子元素运用 `yAxis` 动画，使其沿着 `y` 轴向上移动

  


```CSS
@layer animation {
    @keyframes xAxis {
        50% {
            animation-timing-function: ease-in;
            translate:450px 0;
        }
    }

    @keyframes yAxis {
        50% {
          animation-timing-function: ease-out;
          translate: 0 -450px;
        }
    }
    
    .dot {
        animation: xAxis 2.5s infinite cubic-bezier(0.02, 0.01, 0.21, 1);
    
        &::after {
            animation: yAxis 2.5s infinite cubic-bezier(0.3, 0.27, 0.07, 1.64);
        }
    }
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8999947a50d84af6ab041c1177e7780e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1434&h=590&s=964494&e=gif&f=82&b=e4e4e4)

> Demo 地址：https://codepen.io/airen/full/JjwLYLM

  


如果你在使用 [WAPPI](https://www.w3.org/TR/web-animations-1/) （Web Animations API）开发动画的话，那么你可以使用它的 `composite 功能`来修复示例中动画所存在的问题。

  


首先，你可以使用 WAPPI 来创建与之前 CSS 示例等效的动画：

  


```CSS
const dot = document.querySelector('.dot');

// xAxis Animation
dot.animate({
    transform: ['translateX(450px)']
}, {
    duration: 2500,
    easing: 'cubic-bezier(0.02, 0.01, 0.21, 1)',
    iterations: Infinity
});

// yAxis Animation
dot.animate({
    transform: ['translateY(-450px)']
}, {
    duration: 2500,
    easing: 'cubic-bezier(0.3, 0.27, 0.07, 1.64)',
    iterations: Infinity
});
```

  


默认行为保持不变，因此这段代码将与 CSS 制作的动画效果相同：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef1c3eddeaf6477bb66dd22754b1d3eb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1352&h=628&s=580502&e=gif&f=77&b=e8e8e8)

  


> Demo 地址：https://codepen.io/airen/full/eYbMpQW

  


如果我们将第二个动画添加 `composite` 属性，并且设置其值为 `add` ，它将告诉浏览器，我们告诉它将此动画的值添加到该属性的当前状态中：

  


```CSS
// yAxis Animation
dot.animate({
    transform: ['translateY(-450px)']
}, {
    duration: 2500,
    easing: 'cubic-bezier(0.3, 0.27, 0.07, 1.64)',
    iterations: Infinity,
    composite: 'add'
});
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/30929a0b84514abd9e292b7a6341f394~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1278&h=598&s=612805&e=gif&f=78&b=e7e7e7)

  


> Demo 地址：https://codepen.io/airen/full/GRPxpLa

  


注意，WAPPI 相关知识已超出这节课的范畴，在这里不做过多的阐述，如果感兴趣，可以搜索相关关键词。

  


庆幸的是，CSS 的 [CSS Animations Level 2](https://drafts.csswg.org/css-animations-2/) 也引入了类似 WAPPI 的 `composite` 的功能，即 [CSS 的动画合成功能](https://drafts.csswg.org/css-animations-2/#propdef-animation-composition)（`animation-composite`）。也就是说，我们现在可以直接使用 CSS 动画合成来修复前面的动画，可以不需要再对多动画进行分层了。

  


```CSS
@keyframes xAxis {
    50% {
        animation-timing-function: ease-in;
        translate:450px 0;
    }
}

@keyframes yAxis {
    50% {
        animation-timing-function: ease-out;
        translate:0 -450px;
    }
}

.dot {
    animation: 
        xAxis 2.5s infinite cubic-bezier(0.02, 0.01, 0.21, 1),
        yAxis 2.5s infinite cubic-bezier(0.3, 0.27, 0.07, 1.64);
    animation-composition: add; 
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6a8a0b9415c4cb490af0399a454ebd5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1272&h=564&s=809967&e=gif&f=91&b=eaeaea)

  


> Demo 地址：https://codepen.io/airen/full/poqedYJ

  


换句话说，你现在可以使用分层、WAPPI 和动画合成等技术来运用多动画，并且尽可能的避免多动画带来的弊端：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3236b885329649acb0f064ab9121a311~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1178&h=586&s=549014&e=gif&f=52&b=e9e9e9)

  


> Demo 地址：https://codepen.io/airen/full/YzdawGb

  


课程开头就说了，CSS 动画合成（`animation-composition`）是一个很新的特性，你可能从未接触过，甚至对它感到很陌生。即使如此，也不用担心，接下来我们将一起来探讨该特性。

  


## CSS 动画合成是什么？

  


[W3C 规范是这样描述 CSS 合成动画的](https://drafts.csswg.org/css-animations-2/#animation-composition)：

  


> The `animation-composition` property defines the composite operation used when multiple animations affect the same property simultaneously.

  


大致的意思是说：“`animation-composition` 属性定义了在多个动画同时影响同一属性时所使用的复合操作”。

  


换句话说，CSS 动画合成（`animation-composition`）是指同时使用多个 CSS 动画效果来影响同一属性时，通过一些属性和方法来控制这些动画如何组合影响动画元素的属性值。这使得在多个动画效果同时作用于元素时，可以更加灵活地控制动画的表现方式，包括属性值的叠加、替换等。这为 Web 开发人员提供了更多的控制权，以创建复杂的动画效果，而不必依赖于单一的动画规则。

  


CSS 动画合成通常使用 CSS 的 `animation-composite` 或 WAPPI 的 `composite` 方法来实现。这有助于避免不必要的复杂性和冲突，使多个动画效果可以协同工作，以创建更加复杂和吸引人的动画效果。

  


注意，在这节课中，我们主要以 CSS 的 `animation-composite` 为主。

  


## CSS 动画合成如何使用？

  


前面我们已经向大家展示了，在 CSS 中，如何使用 CSS 合成动画 `animation-composition` 来解决多动画存在的问题。也就是说，**CSS 的** **`animation-composition`** **属性用于定义当多个动画同时影响同一属性时的复合操作**。例如：

  


```CSS
@keyframes move {
    0% {
        transform: translate(0, 0);
    }
    50% {
        transform: translate(100px, 0);
    }
    100% {
        transform: translate(0, 0);
    }
}

@keyframes rotate {
    0% {
        transform: rotate(0deg);
    }
    100% {
        transform: rotate(360deg);
    }
}

.element {
    animation-name: move, rotate;
    animation-duration: 2s;
    animation-timing-function: ease-in-out;
    animation-iteration-count: infinite;
    animation-composition: add;
}
```

  


上面这个示例中，元素 `.element` 同时应用了 `move` 和 `rotate` 两个动画，并且显式地设置了 `animation-composition` 属性的值为 `add` 。这意味着 `move` 和 `rotate` 两个动画效果将相加，而不是互相替换。这样，`.element` 元素将同时执行 `move` 和 `rotate` 动画，而不是先执行 `move` 动画，再执行 `rotate` 动画。

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2fefc25b6dd425195ac943089fff478~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=990&h=484&s=311519&e=gif&f=76&b=4a4969)

  


> Demo 地址：https://codepen.io/airen/full/wvRmGEx

  


你可以根据需要将 `animation-composition` 设置为不同的值，以控制动画之间的复合方式。该属性可选的值包括 `replace` 、`add` 和 `accumulate` ，每个选项都会产生不同的效果：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8cba7e6877bf4f849c8cd0c5e724e7d8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1432&h=498&s=1134834&e=gif&f=70&b=4a4969)

  


> Demo 地址：https://codepen.io/airen/full/KKbozbY

  


正如你所看到的：

  


-   `animation-composition` 取值为 `replace` 时，元素只有旋转效果
-   `animation-composition` 取值为 `add` 时，元素有位移加放置的效果
-   `animation-composition` 取值为 `accumulate` 时，元素只有位移效果

  


所以说，理解复合操作可以帮助你更精细地控制多个动画之间的互动效果。

  


## 如何理解 CSS 的动画合成

  


我们通过下面这个示例来向大家阐述 CSS 的动画合成是如何工作的，即如何理解动画合成。

  


假设，你在一个 `.animated` 元素上应用了下面这段 CSS 代码：

  


```CSS
.element {
    filter: blur(5px);
    animation: filter 2s linear infinite;
}

@keyframes filter {
    0% {
        filter: blur(10px);
    }
    100% {
        filter:blur(20px);
    }
}
```

  


就上面代码而言，初始状态下，用于 `.element` 元素上 `filter` 属性的 `blur(5px)` ，被称为 `filter` 底层值，而 `@keyframes` 中第一帧的 `filter` 属性的 `blur(10px)` 被称为效果值。`animation-composition` 属性指定在合成底层值和效果值的效果之后执行的操作，以产生最终的效果值。`.element` 元素上设置不同的 `animation-composition` 值将会影响 `filter` 属性的最终值，从而影响最终的效果。

  


先来看 `replace` 值，它是 它是 `animation-composition` 的默认值。当 `.element` 元素的 `animation-composition` 属性的值为 `replace` 时，那么 `@keyframes` 中第一帧（`0%`）的 `blur(10px)` （效果值）将会替换 `.element` 元素的底层值 `blur(5px)` 。此时，元素 `.element` 是从 `blur(10px)` 到 `blur(20px)` 的一个动画效果。即：

  


```CSS
@keyframes filterReplace {
    0% {
        filter: blur(10px);
    }
    100% {
        filter: blur(20px);
    }
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56a15091b6f446f79b7b74d25cb813a6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=986&h=498&s=2719776&e=gif&f=99&b=4a4969)

  


> Demo 地址：https://codepen.io/airen/full/qBLoNGK

  


如果元素 `.element` 的 `animation-composition` 属性的值是 `add` ，那么 `0%` 关键帧中的 `filter` 属性的值（效果值，即 `blur(10px)`）和元素 `filter` 属性的底层值（`blur(5px)`）会叠加在一起。元素 `.element` 的 `filter` 属性复合值将为 `blur(5px) blur(10px)` 。它相当于：

  


```CSS
@keyframes filterAdd {
    0% {
        filter: blur(5px) blur(10px);
    }
    100% {
        filter: blur(20px);
    }
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48c5656f43d24a1782838d8c60961a04~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=990&h=494&s=4217235&e=gif&f=138&b=4a4969)

  


> Demo 地址：https://codepen.io/airen/full/VwqXjoq

  


如果元素 `.element` 的 `animation-composition` 属性的值是 `accumulate`，那么 `0%` 关键帧中的 `filter` 属性的复合效果值将为 `blur(15px)` 。它相当于：

  


```CSS
@keyframes filterAccumulate {
    0% {
        filter: blur(15px)
    }
    
    100% {
        filter: blur(20px);
    }
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/552e4986f45d4d5583279a28afbaef0a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=992&h=500&s=2823856&e=gif&f=95&b=4a4969)

  


> Demo 地址：https://codepen.io/airen/full/KKbogVW

  


简单地对 `animation-composition` 属性值含义做个描述：

  


-   **`replace`** **（替换）** ：这是 `animation-composition` 属性的默认行为。当多个动画同时影响同一个属性时，最后一个动画会完全替换之前的动画效果。换句话说，只有最后一个动画的效果会应用到属性上。例如前面示例中的 `rotate` 动画完全替代了 `move` 动画，所以最终只有旋转的动画效果。
-   **`add`** **（叠加）** ：使用 `add` 复合操作时，多个动画的效果会叠加在一起。这意味着它们会一起影响属性的最终值。例如，如果一个动画元素向右移动 `30px` ，而另一个动画使元素向左移动 `20px` ，使用 `add` 复合操作之后，元素将向右移动 `10px`
-   **`accumulate`** **（累积）** ：使用 `accumulate` 复合操作时，动画效果会逐渐累积在一起，而不是替换或叠加。这意味着每个动画都会根据之前动画的效果来计算最终效果。例如上面的滤镜示例，元素默认有一个 `blur(5px)` ，`filter` 动画的 `0%` 位置有一个 `blur(10px)` ，使用 `accumulate` 复合操作后，`0%` 关键帧的 `filter` 属性的复合值是 `blur(15px)`

  


由于我们平时制作 CSS 动画时常会用到 CSS 的 `transform` 属性，所以我想再花一点时间，以 `transform` 为例，以不同方式的案例向大家阐述动画合成是如何工作的。

  


先来看一个简单的 `transform` 示例。假设你在一个 `.element` 元素上运用了下面这段 CSS 代码：

  


```CSS
.element {
    transform-origin: 50% 50%;
    transform: translateX(50px) rotate(45deg);
}
```

  


同时你设置了一个 `@keyframes` ：

  


```CSS
@keyframes adjust {
    to {
        transform: translateX(100px);
    }
}
```

  


如果你将 `adjust` 动画应用于元素 `.element` 时，那么这些关键帧将会应用于元素。因此，`adjust` 中的 `to` 关键帧中的 `transfrom` 将会替代元素 `.element` 初始设置的 `transform` 。这和前面那个滤镜示例是相似的，这也是浏览器的默认行为：

  


```CSS
@keyframes adjust {
    to {
        transform: translateX(100px);
    }
}

.element {
    transform-origin: 50% 50%;
    transform: translateX(50px) rotate(45deg);
    animation: adjust 5s linear infinite alternate;
}
```

  


你在浏览器中看到的效果如下图所示：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ef2fc787ef3478cb8b9bbdb441e3d44~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=980&h=496&s=499440&e=gif&f=95&b=f7f7f7)

  


> Demo 地址：https://codepen.io/airen/full/RwEMGYW

  


上图中，浅蓝色的正方形是 `.element` 在没有使用 `adjust` 动画时的效果（初始状态效果），它沿着 `x` 轴向右移动了 `50px` （即 `translateX(50px)`），然后旋转了 `45deg` （即 `roate(45deg)`）。图中深蓝色正方形是 `.element` 元素使用了 `adjust` 动画之后的效果，`adjust` 中的 `to` 关键帧中的 `transform` 替代了现有的 `transform` 。如果你只让动画播放一次，并将 `animation-fill-mode` 设置为 `forwards` 或 `both` ，你就能更清楚的看到 `to` 关键帧中的 `transform` 替代元素现有的 `transform` ，即 `translateX(50px)` 替代了 `translateX(50) rotate(45deg)` ：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3f51f6f3d5c4b84a1de1824f73829da~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2000&h=1157&s=576814&e=jpg&b=f1f1f1)

  


> Demo 地址：https://codepen.io/airen/full/XWoENKw

  


其实，你现在看到的效果与 `animation-composition` 属性的 `replace` 值的效果等同。你可以使用 `add` 或 `accumulate` 值来替代默认的 `replace` 值。

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/794bfed280c24515a5034abb5ee5e367~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1428&h=354&s=2995060&e=gif&f=205&b=dadada)

  


> Demo 地址：https://codepen.io/airen/full/QWzmGwm

  


稍加解释一下。

  


第一个很简单，`replace` 会使 `to` 关键词中的 `transform` 替换元素 `element` 底层的 `transform` ，即 `translateX(100px)` 替代了 `translateX(50px) rotate(45deg)` ：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ca84b6cc4c24ab98aa4661d97a7162c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2000&h=1358&s=711161&e=jpg&b=f3f3f3)

  


`animation-composition` 属性的 `add` 值会将 `to` 关键词中的 `translateX(100px)` （即效果值）添加到 `.element` 元素的 `transform` 属性中。也就是说，元素 `.element` 最后的 `transform` 的值变成了 `translateX(50px) rotate(45deg) translateX(100px)` ，即在 `to` 关键词中，元素会先执行 `translateX(50px)` ，再执行 `rotate(45deg)` ，最后执行 `translateX(100px)` ：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/13b0de2aa15a4a74b2de379a1ba6797e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2000&h=1592&s=949959&e=jpg&b=f0f0f0)

  


同样的，你可以尝试着将动画只播放一次，并且让元素停留在动画播放完的位置，你会发现 `animation-composition` 取值为 `add` 时，元素 `.element` 的 `transform` 属性的最终效果值：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3212e186b1384a1585fe05095d6a1a9b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1018&h=396&s=1267680&e=gif&f=163&b=efefef)

  


> Demo 地址：https://codepen.io/airen/full/zYyWZxR

  


`animation-composition` 属性的 `accumulate` 值会将 `to` 关键词中 `transform` 属性的值 `translateX(100px)` 与元素 `.element` 的 `transform` 属性的 `translateX(50px) rotate` 相结合，即可效果值与底层值相结合。你可以理解成，动画混合后，元素会先执行 `translateX(150px)` ，然后再执行 `rotate(45deg)` ：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0cd3802770b54c6d934638d779abb560~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2000&h=1907&s=964940&e=jpg&b=ededed)

  


> Demo 地址：https://codepen.io/airen/full/bGOvqqe

  


你也可以但看动画的 `to` 关键词状态下的效果：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9c8cbce62144056a6a57833278490bb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1002&h=410&s=1797994&e=gif&f=198&b=f0f0f0)

  


> Demo 地址：https://codepen.io/airen/full/abPYJYQ

  


你可以将 `animation-composition` 类比为一杯被茶填满的情况。当倒入牛奶时会发生以下情况：

  


-   `replace`：茶被移除，并被牛奶替换
-   `add`：牛奶被添加到杯子中，但仍然位于茶的顶部
-   `accumulate`：牛奶被加到茶中，因为它们都是液体，所以它们会很好地混合在一起

  


现来看一个更为复杂一点的动画。我们把上面示例中的 `adjust` 帧动画调整为像下面这样：

  


```CSS
@keyframes adjust {
    20%,
    40% {
        transform: translateX(100px);
    }
    80%,
    100% {
        transform: translateX(150px);
    }
}

.element {
    transform-origin: 50% 50%;
    transform: translateX(50px) rotate(45deg);
    animation: adjust 5s linear infinite alternate;
    
    &.replace {
        animation-composition: replace;
    }
    &.add {
        animation-composition: add;
    }
    &.accumulate {
        animation-composition: accumulate;
    }
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3b3f3e43b75421a8b67087cf77fbe51~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1426&h=354&s=1871020&e=gif&f=161&b=dadada)

  


> Demo 地址：https://codepen.io/airen/full/NWeYpEg

  


正如你所看到的：

  


-   `animation-composition` 属性使用 `replace` ，`adjust` 动画的 `20%` 和 `40%`关键帧中 `transform` 属性的最终效果值是 `translateX(100px)`（完全替换了底层值 `translateX(50px) rotate(45deg)`）。在这种情况下，元素从 `45deg` 旋转到 `0deg`，因为它从 `.element` 元素本身设置的默认值（`transform: translateX(50px) rotate(45deg)`）动画到 `20%` 标记处设置的非旋转值（`transform: translateX(100%)`）。
-   `animation-composition` 属性使用 `add`，`adjust` 动画的 `20%` 和 `40%` 关键帧中 `transform` 属性的最终效果值是 `translateX(50px) rotate(45deg)`，然后是 `translateX(100px)`。因此，`.element` 元素先向右移动 `50px`，旋转 `45deg`，然后沿着 `x` 轴再向右移动 `100px`。
-   `animation-composition` 属性使用 `accumulate`，`adjust` 动画的 `20%` 和 `40%` 关键帧中的最终效果值是 `translateX(150px) rotate(45deg)`。这意味着两个 `x` 轴平移值 `50px` 和 `100px` 被合并或“累积”在一起。

  


上面我们所展示的都是 `animation-composition` 属性设置单个值的，其实它也可以设置以逗号分隔开来的多个值，例如：

  


```CSS
.element {
    animation-composition: add, accumulate;
}
```

  


那么，动画合成 `animation-composition` 设置多个值时，它又是如何工作呢？在回答该问题之前，我们有必要先给点时间了解一下 CSS 的多动画。

  


CSS 的 `animation` 属性以及它的子属性（`animation-*`）可以接受多个逗号分隔的值，这就是多动画的设置。例如：

  


```CSS
.multi-animation {
    animation: fadeIn 2s linear, fadeOut 1s linear 1s;
}

.multi-animation {
    animation-name: fadeIn, move, bounce;
    animation-duration: 2s, 1s, 1s,
    animation-timing-function: linear, ease;
}

.multi-animation {
    animation-name: fadeOut, bounce, rotate;
    animation-duration: 2s;
}
```

  


代码中的示例都是多动画的正确使用方式。

  


也就是说，当你希望在单个规则中应用多个动画并为每个动画设置不同的参数时，比如持续时间，缓动函数，迭代次数等，可以使用此功能。接下来，我将通过几个简单的示例，来解释多个值不同的排列方式。

  


先来看第一个示例：

  


```CSS
.multi-animation {
    animation-name:fadeIn, rotateOut, bounceIn;
    animation-duration: 1s, 2s, 1s;
    animation-timing-function: linear, ease, ease-in-out;   
}
```

  


在这个示例中，`.multi-animation` 元素同时应用了名为 `fadeIn` 、`rotateOut` 和 `bounceIn` 的帧动画，并且每个动画的持续时间和缓动函数都不一样。

  


-   `fadeIn` 动画持续播放 `1s` ，对应的缓动函数是 `linear`
-   `rotateOut` 动画持续播放 `2s` ，对应的缓动函数是 `ease`
-   `bounceIn` 动画持续播放 `1s` ，对应的缓动函数是 `ease-in-out`

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/244f8b2edb2b4cb0b9283351a37a52d2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=842&h=462&s=588714&e=gif&f=180&b=ededed)

  


> Demo 地址：https://codepen.io/airen/full/ZEVxyaO

  


我们还可以像下面这样设置多个动画：

  


```CSS
.multi-animation {
    animation-name:fadeIn, rotateOut, bounceIn;
    animation-duration: 1s;
    animation-timing-function: linear;   
}
```

  


在这个示例中，我们同样使用 `animation-name` 属性给 `.multi-animation` 元素设置了 `fadeIn` 、`rotateOut` 和 `bounceIn` 三个动画，但三个动画只有一个持续时间和缓动函数。在这种情况之下，`fadeIn` 、`rotateOut` 和 `bounceIn` 三个动画都会被赋予相同的持续时间（`1s`）和缓动函数（`linear`）：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2837e3c382146528e20ccfc6831900a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=860&h=456&s=594261&e=gif&f=176&b=ededed)

  


> Demo 地址：https://codepen.io/airen/full/YzdaJWq

  


把上面示例代码调整成下面这样：

  


```CSS
.multi-animation {
    animation-name:fadeIn, rotateOut, bounceIn;
    animation-duration: 1s, 2s;
    animation-timing-function: linear, ease-in-out;   
}
```

  


上面代码，依旧通过 `animation-name` 属性给 `.multi-animation` 指定了三个动画，即 `fadeIn` 、`rotateOut` 和 `bounceIn` ，但持续时间和缓动函数只有两个。在这种情况下，如果列表中没有足够的值来为每个动画分配单独的值，那么值分配将从可用列表中的第一个项目循环到最后一个项目，然后重新循环到第一个项目。因此：

  


-   `fadeIn` 动画的持续时间是 `1s` ，缓动函数是 `linear`
-   `rotateOut` 动画的持续时间是 `2s` ，缓动函数是 `ease-in-out`
-   `bounceIn` 动画的持续时间是 `1s` ，级动函数是 `linear`

  


这是因为 `rotateOut` 动画获得的持续时间 `2s` 和缓动函数 `ease-in-out` 分别是 `animation-duration` 和 `animation-timing-function` 属性的值列表中的最后一个。因此，持续时间和缓动函数的值分配将重置为第一个值，所以 `bounceIn` 动画的持续时间是 `1s` ，缓动函数是 `linear` 。

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7bafb328a771433f974411a21297ad9f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=962&h=468&s=640214&e=gif&f=166&b=eeeeee)

  


> Demo 地址：https://codepen.io/airen/full/eYbMPBx

  


如果动画数量与动画属性值的不匹配是反过来的，比如：

  


```CSS
.multi-animation {
    animation-name:fadeIn, rotateOut, bounceIn;
    animation-duration: 1s, 2s, .2s, .4s, .5s;
    animation-timing-function: linear, ease-in-out, ease, ease-in, ease-out;   
}
```

  


上面代码中，`animation-name` 属性只显式指定了三个动画，但 `animation-duration` 和 `animation-timing-function` 分别指定了五个持续时间和缓动函数，比动画名称多出两个。在这种情况下，多出来的持续时间（即 `.4s` 和 `.5s`）和缓动函数（即 `ease-in` 和 `ease-out` ）将不会应用于任何动画，并且会被忽略。

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f4146a9144ab4e48a91c2cf42d28134f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=990&h=430&s=550340&e=gif&f=146&b=f0f0f0)

  


> Demo 地址：https://codepen.io/airen/full/MWZVPoZ

  


另外，当同一个元素应用多个动画时，它们将按照出现在 `animation-name` 的顺序来执行，即从左往右执行。比如下面这个示例：

  


```CSS
@keyframes slideInLeft {
  from {
    transform: translate3d(-100vw, 0, 0);
    visibility: visible;
  }

  to {
    transform: translate3d(0, 0, 0);
  }
}

@keyframes swing {
  20% {
    transform: rotate3d(0, 0, 1, 15deg);
  }

  40% {
    transform: rotate3d(0, 0, 1, -10deg);
  }

  60% {
    transform: rotate3d(0, 0, 1, 5deg);
  }

  80% {
    transform: rotate3d(0, 0, 1, -5deg);
  }

  to {
    transform: rotate3d(0, 0, 1, 0deg);
  }
}

@keyframes slideOutRight {
  from {
    transform: translate3d(0, 0, 0);
  }

  to {
    visibility: hidden;
    transform: translate3d(100vw, 0, 0);
  }
}

.multi-animation {
    animation-name: slideInLeft, swing, slideOutRight;
    animation-duration: 1s, 2s, 1s;
    animation-delay: 0s, .5s, 2.5s;
    animation-timing-function: ease-in-out;
    transform-origin: top center;
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6b96528ea6148349abbbd78c9b89d6f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=982&h=478&s=805744&e=gif&f=264&b=efefef)

  


> Demo 地址：https://codepen.io/airen/full/abPYREb

  


正如你所看到的，它会先执行 `slideInLeft` 动画，然后是 `swing` 动画，最后才是 `slideOutRight` 动画。如果其他属性值不变，只调整 `animation-name` 属性值，你会发现，应用于元素的动画效果又将会是不一样。例如：

  


```CSS
.multi-animation {
    animation-name: swing, slideInLeft, slideOutRight;
}
```

  


上面代码把 `swing` 动画调整到最左侧了，此时，元素会先执行 `swing` 动画，再执行 `slideInLeft` 动画，最后才执行 `slideOutRight` 动画：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6874d771e3a243b384e7a3f0a1b28db9~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=940&h=468&s=672437&e=gif&f=240&b=eeeeee)

  


> Demo 地址：https://codepen.io/airen/full/qBLoJob

  


注意，如果有多个动画应用在同一个元素上，并且它们修改了相同的属性，最后一个动画的效果将覆盖之前的动画。因此，动画的顺序很重要。

  


我想你现在对于多个动画的应用有了一定的了解，那么我们就可以接着聊多个动画合成了。多个动画合成指的是 `animation-composition` 属性的值是一个列表值，例如：

  


```CSS
.element {
    animation-composition: add, replace, accumulate;
}
```

  


它同样遵循多个动画运用的规则。先来看第一个示例：

  


```CSS
.multi-animation {
    animation-name:fadeIn, rotateOut, bounceIn;
    animation-duration: 1s, 2s, 1s;
    animation-timing-function: linear, ease, ease-in-out;   
    animation-composition: add, replace, accumulate;
}
```

  


上面代码中，`animation-composition` 属性值列表数量与 `animation-name` 属性值列表数量相同，动画合成的值与动画是一一对应的关系：

  


-   `fadeIn` 动画应用了 `add` （叠加）
-   `rotateOut` 动画应用了 `replace` （替代）
-   `bounceIn` 动画应用了 `accumulate` （累积）

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/739a5774480b4f938ace79d3b142a5e8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=996&h=464&s=561880&e=gif&f=150&b=eeeeee)

  


> Demo 地址：https://codepen.io/airen/full/ExGEdpa

  


再来看第二个示例，动画名称有三个，动画合成只使用了一个：

  


```CSS
.multi-animation {
    animation-name:fadeIn, rotateOut, bounceIn;
    animation-duration: 1s;
    animation-timing-function: linear;  
    animation-composition: add; 
}
```

  


这个很好理解，它会告诉浏览器，应用在元素 `.multi-animation` 上的 `fadeIn` 、`rotateOut` 和 `bounceIn` 三个动画的合成方式都是 `add` （叠加）：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38dd8ecff5b24a0d9c42cd67a4ebf89a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=836&h=458&s=499284&e=gif&f=135&b=ececec)

  


> Demo 地址：https://codepen.io/airen/full/LYMdgJZ

  


再来看第三个示例：

  


```CSS
.multi-animation {
    animation-name:fadeIn, rotateOut, bounceIn;
    animation-duration: 1s, 2s;
    animation-timing-function: linear, ease-in-out;   
    animation-composition: add, accumulate;
}
```

  


应用于元素的动画合成数量少于动画名称，它将会从第一个值开始循环：

  


-   `fadeIn` 动画应用的是 `add` （叠加）
-   `rotateOut` 动画应用的是 `accmulate` （累积）
-   `bounceIn` 动画应用的是 `add` （叠加），因为 `animation-composition` 属性的值列表只有两个值，`rotateOut` 动画已经用完其第二个值，因此 `bounceIn` 动画会从值列表的第一个值开始循环使用

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/691a10cdd21d4b3db9b8fb462bc8f5bc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=926&h=462&s=726692&e=gif&f=230&b=eeeeee)

  


> Demo 地址：https://codepen.io/airen/full/MWZVPPR

  


同样的，如果应用于 `animation-composition` 属性值列表数量多于动画名称的数量，那么多出来的值将被忽略：

  


```CSS
.element.playing {
    animation-name:fadeIn, rotateOut, bounceIn;
    animation-duration: 1s, 2s,.3s,.4s,.5s;
    animation-timing-function: linear, ease-in-out,ease, ease-in, ease-out;   
    animation-composition: add, replace, accumulate, replace, add;
}
```

  


-   `fadeIn` 动画应用的是 `add`
-   `rotateOut` 动画应用的是 `replace`
-   `bounceIn` 动画应用的是 `accumulate`

  


`animation-composition` 属性列表值中多出来的 `replace` 和 `add` （从右往左数的两个）则直接被忽略：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96f6901cc73e486f864618ee62014567~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1042&h=582&s=918851&e=gif&f=170&b=ebebeb)

  


> Demo 地址：https://codepen.io/airen/full/xxmWymZ

  


所以说，多个动画合成的使用和多动画的使用是一样的。也就是说，当你在 `animation-*` 属性上指定多个逗号分隔的值时，它们将按照 `animation-name` 出现的顺序应用于动画。**如果动画数量和合成数量不同，** **`animation-composition`** **属性中列出的值将循环从第一个** **`animation-name`** **到最后一个** **`animation-name`** **，直到所有动画都有分配的** **`animation-composition`** **值**。

  


在课程中，我们多次提到，多个动画应用在同一元素上，并且它们修改了相同的属性，将以最后一个动画中的属性为准。因此，动画的顺序很重要。可是，随着 CSS 动画合成的出现，它将打破这一规则，前面我们也花了很大的篇幅阐述了动画合成是如何影响元素的动画效果的。其实，元素应用多个动画合成，它的基本原理是一样的。为了更易于帮助大家理解多个动画合成是如何影响元素动画效果的，我这里尽可能把示例简单化。例如，你首先在元素上应用了下面这段代码：

  


```CSS
.element {
    transform: translateX(50px);
}
```

  


有一个简单的位移，元素沿着 `x` 轴向右平移了 `50px` ，这个大家都懂。现在，你给 `.element` 元素添加了下面三个简单的动画：

  


```CSS
@keyframes moveX {
    to {
        transform: translateX(100px);
    }
}

@keyframes rotate {
    to {
        transform: rotate(45deg);
    }
}

@keyframes moveY {
    to {
        transform: translateY(200px);
    }
}

.element {
    animation: 
        moveX 1s linear infinite alternate,
        rotate 1s linear infinite alternate,
        moveY 1s linear infinite alternate;
    animation-composition: add, replace, accumulate;
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ad40bfbbe614a2f8f9922d25d5ea99e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=990&h=414&s=322885&e=gif&f=40&b=efefef)

  


> Demo 地址：https://codepen.io/airen/full/GRPxYVQ

  


我将用下图来拆分一下整个动画效果：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/048c4a06ac47476682b80f22cb1eae58~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2000&h=1907&s=1361522&e=jpg&b=f2f2f2)

  


简单解释一下：

  


-   元素默认设置了一个 `transform` ，其值是 `translateX(50px)` ，即上图中浅蓝色的正方形，它从红色虚线框向右移动了 `50px` 。其中 `translateX(50px)` 是 `transform` 的底层值
-   元素设置了三个动画 `moveX` 、`rotate` 和 `moveY` ，这三个动画的 `to` 关键帧都重新对 `transform` 属性设置了值，分别是 `translateX(100px)` 、`rotate(45deg)` 和 `translateY(200px)` 。它们的执行顺序是 `moveX` ，然后 `rotate` ，最后才是 `moveY`
-   使用 `animation-composition` 分别给 `moveX` 、`rotate` 和 `moveY` 动画设置了合成模式，对应的是 `add` （叠加）、`replace` （替代）和 `accumulate` （混合）
-   由于 `moveX` 动画的合成模式是 `add` （叠加），该动执行到 `to` 关键帧（最后一帧）时，对应的 `transform` 会与元素底层的 `transform` 相叠加，此时 `transform` 的效果值变成 `translateX(50px) translateY(100px)`
-   接着元素会执行 `rotate` 动画，由于该动画设置的合成模式是 `replace` （替换），所以动画执行到 `to` 关键帧时，对应的 `transform` 的值会直接替换 `moveX` 动画合成后的 `transform` 效果值，此时 `transform` 的效果值就变成 `rotate(45deg)`
-   最后执行的是 `moveY` 动画，该动画设置的合成模式是 `accumulate` （混合），动画执行到 `to` 关键帧时，对应的 `transform` 的值会与 `rotate` 动画合成后的 `transform` 值相混合，此时 `transform` 的效果值就变成 `translateY(200px) rotate(45deg)`

  


你可以尝试着将所有动画只播放一次，并且状态停留在结束位置，能看到最终结果与分解所描述的结果是一致的，如下图所示：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38b3d64375a24200af2a79a4b8fdf493~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1038&h=648&s=1405227&e=gif&f=94&b=e9e9e9)

  


> Demo 地址：https://codepen.io/airen/full/YzdaRzr

  


如此一来，你可以更进一步的使用 CSS 的 `animation-composition` 来控制动画，制作出效果更佳的动画。

  


## CSS 动画合成的用例

  


我想阅读到这里，再回过头来看课程开头的示例，你应该明白了动画合成的功能与作用。这里我还是要再重复一次：

  


> CSS 动画合成 `animation-composition` 属性用于确定当多个动画同时影响相同属性时应该发生什么情况。它主要用于控制多个动画如何组合它们的效果，特别是当它们影响相同的 CSS 属性时。

  


我们在理解和使用 CSS 动画合成的时候，需要避免一个误区。这个误区是，CSS 动画合成并不是把多个动画合在一起，而是多个元素应用到同一个元素，并且影响同一个属性时，它是如何改变元素的属性。这对于具有相同属性的多个动画非常有用，可以根据需求选择不同的组合方式，以实现所需的动画效果。

  


记得在小册《[CSS 变换之单个变换](https://juejin.cn/book/7223230325122400288/section/7259668493158023205)》的课程中，我曾举过一个给模态框添加动画效果的案例。通常情况之下，我们一般会使用下面的 CSS 代码让模态框在视窗中水平垂直居中：

  


```CSS
.modal {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
}
```

  


但为了给模态框添加动画效果，可能会在 `@keyframes` 中用到元素的 `transform` 属性，比如：

  


```CSS
@layer modal {
    @keyframes bounceInDown {
        from,
        60%,
        75%,
        90%,
        to {
            animation-timing-function: cubic-bezier(0.215, 0.61, 0.355, 1);
        }
    
        0% {
            opacity: 0;
            transform: translate3d(0, -3000px, 0) scaleY(3);
        }
    
        60% {
            opacity: 1;
            transform: translate3d(0, 25px, 0) scaleY(0.9);
        }
    
        75% {
            transform: translate3d(0, -10px, 0) scaleY(0.95);
        }
    
        90% {
            transform: translate3d(0, 5px, 0) scaleY(0.985);
        }
    
        to {
            transform: translate3d(0, 0, 0);
        }
    }
    
    dialog {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        animation: bounceInDown 0.28s cubic-bezier(0.215, 0.61, 0.355, 1) both;
    }
}
```

  


你会发现，添加 `bounceInDown` 动效之后的模态框，在动效结束时，它的位置也被改变了，并没有在浏览器视窗中水平居中：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2a4b26f3a4b74b0f989f514db3f878ef~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1070&h=622&s=3177372&e=gif&f=114&b=6f6e91)

  


这是因为运用于模态框的 `transform` 并不是最初设置的值（`transform: translate(-50%,-50%)`），而是被 `@keyframes` 中最后一帧的 `transform` 属性值（`translate3d(0,0,0)`）覆盖了。如果要让添加了 `bounceInDown` 动效的模态框，在动效结束之后依旧在浏览器视窗中水平垂直居中，我们不得不改变水平垂直居中的布局方案，或者调整 `bounceInDown` 动画中每一帧的 `transform` 的值，例如：

  


```CSS
@layer modal {
    @keyframes bounceInDown {
        from,
        60%,
        75%,
        90%,
        to {
            animation-timing-function: cubic-bezier(0.215, 0.61, 0.355, 1);
        }
    
        0% {
            opacity: 0;
            transform : translate3d (- 50% , calc (- 3000px - 50% ), 0 ) scaleY ( 3 );
        }
    
        60% {
            opacity: 1;
            transform : translate3d (- 50% , calc ( 25px - 50% ), 0 ) scaleY ( 0.9 );
        }
    
        75% {
            transform : translate3d (- 50% , calc (- 10px - 50% ), 0 ) scaleY ( 0.95 );
        }
    
        90% {
            transform : translate3d (- 50% , calc ( 5px - 50% ), 0 ) scaleY ( 0.985 );
        }
    
        to {
            transform : translate3d (- 50% , - 50% , 0 );
        }
    }
    
    dialog {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        animation: bounceInDown 0.28s cubic-bezier(0.215, 0.61, 0.355, 1) both;
    }
}
```

  


> Demo 地址：https://codepen.io/airen/full/rNoOMWL

  


或者我们使用单个变换属性来替代 `transform` 属性：

  


```CSS
@layer modal {
    @keyframes bounceInDown {
        from,
        60%,
        75%,
        90%,
        to {
            animation-timing-function: cubic-bezier(0.215, 0.61, 0.355, 1);
        }
    
        0% {
            opacity: 0;
            /* transform: translate3d(-50%, calc(-3000px - 50%), 0) scaleY(3); */
            translate: -50% calc(-3000px - 50%);
            scale: 1 3 1;
        }
    
        60% {
            opacity: 1;
            /* transform: translate3d(-50%, calc(25px - 50%), 0) scaleY(0.9); */
            translate: -50% calc(25px - 50%);
            scale: 1 .9 1;
        }
    
        75% {
            /*transform: translate3d(-50%, calc(-10px - 50%), 0) scaleY(0.95); */
            translate: -50% calc(-10px - 50%);
            scale: 1 .95 1;
        }
    
        90% {
            /* transform: translate3d(-50%, calc(5px - 50%), 0) scaleY(0.985); */
            translate: -50% calc(5px - 50%);
            scale: 1 .985 1;
        }
    
        to {
            /*transform: translate3d(-50%, -50%, 0); */
            translate: -50% -50%; 
        }
    }
    dialog {
        position: absolute;
        top: 50%;
        left: 50%;
        /* transform: translate(-50%, -50%); */
        translate: -50% -50%;
        animation: bounceInDown 0.28s cubic-bezier(0.215, 0.61, 0.355, 1) both;
    }
}
```

  


使用单个变换属性也能获得相同的模态框效果：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b945c527e9e4482a48f9daf344d3a07~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1070&h=514&s=3678850&e=gif&f=139&b=676689)

  


> Demo 地址：https://codepen.io/airen/full/abPYXZg

  


有了 CSS 的动画合成之后，你又多了一种选择：

  


```CSS
@layer modal {
    @keyframes bounceInDown {
        from,
        60%,
        75%,
        90%,
        to {
            animation-timing-function: cubic-bezier(0.215, 0.61, 0.355, 1);
        }
    
        0% {
            opacity: 0;
            transform: translate3d(0, -3000px, 0) scaleY(3);
        }
    
        60% {
            opacity: 1;
            transform: translate3d(0, 25px, 0) scaleY(0.9);
        }
    
        75% {
            transform: translate3d(0, -10px, 0) scaleY(0.95);
        }
    
        90% {
            transform: translate3d(0, 5px, 0) scaleY(0.985);
        }
    
        to {
            transform: translate3d(0, 0, 0);
        }
    }
    
    dialog {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        animation: bounceInDown 0.28s cubic-bezier(0.215, 0.61, 0.355, 1) both;
        animation-composition: add;
    }
}
```

  


上面的代码我们没有对 `bounceInDown` 动画中每一帧的 `transform` 属性进行调整，只是在 `dialog` 元素上新增了 `animation-composition` 属性，并且指定其值为 `add` 。最终效果也如你所期待的一样：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c19f821d3c9345028cefdaf18863a29c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=966&h=542&s=1851039&e=gif&f=87&b=6b6a8d)

  


> Demo 地址：https://codepen.io/airen/full/yLGKZMR

  


注意，在这个示例中，你将 `animation-composition` 属性的值改成 `accumulate` 也能得到同样的效果。

  


你甚至还可以在上例的基础上，你还可以给模态框添加新的动画，比如 `swing`：

  


```CSS
@keyframes swing {
    20% {
        transform: rotate3d(0, 0, 1, 15deg);
    }

    40% {
        transform: rotate3d(0, 0, 1, -10deg);
    }

    60% {
        transform: rotate3d(0, 0, 1, 5deg);
    }

    80% {
        transform: rotate3d(0, 0, 1, -5deg);
    }

    to {
        transform: rotate3d(0, 0, 1, 0deg);
    }
}

dialog {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    transform-origin: top center;
    animation: 
      bounceInDown 0.28s cubic-bezier(0.215, 0.61, 0.355, 1) both,
      swing .2s cubic-bezier(0.215, 0.61, 0.355, 1) .28s both;
    animation-composition: add;
}
```

  


此时，打开模态框的动画效果又将会是另一种：

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbadc0f344e84e819fe84db3a6122000~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=960&h=486&s=3376777&e=gif&f=125&b=69678a)

  


> Demo 地址：https://codepen.io/airen/full/vYvRbJy

  


正如你看到的，你现在可以将 [Animate.css](https://animate.style/) 提供 CSS 动画效果随意组合性的用于动画元素上。最后再来看一个示例，我将 `zoomIn` 、`lightSpeedInLeft` 和 `heartBeat` 三个动画用于模态框上：

  


```CSS
@layer modal {
    @keyframes zoomIn {
        from {
            opacity: 0;
            transform: scale3d(0.3, 0.3, 0.3);
        }
    
        50% {
            opacity: 1;
        }
    }
    
    @keyframes heartBeat {
        0% {
            transform: scale(1);
        }
    
        14% {
            transform: scale(1.3);
        }
    
        28% {
            transform: scale(1);
        }
    
        42% {
            transform: scale(1.3);
        }
    
        70% {
            transform: scale(1);
        }
    }
    
    @keyframes lightSpeedInLeft {
        from {
            transform: translate3d(-100%, 0, 0) skewX(30deg);
            opacity: 0;
        }
    
        60% {
            transform: skewX(-20deg);
            opacity: 1;
        }
    
        80% {
            transform: skewX(5deg);
        }
    
        to {
            transform: translate3d(0, 0, 0);
        }
    }
    
    dialog {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        transform-origin: top center;
        animation: 
            lightSpeedInLeft 0.28s ease-out both, 
            zoomIn 0.2s linear both,
            heartBeat 1.3s cubic-bezier(0.215, 0.61, 0.355, 1) 0.28s both;
        animation-composition: add, accumulate, add;
    }
}
```

  


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d412cd4796b64667a4c6ba218a394bd3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=938&h=492&s=6217560&e=gif&f=173&b=6e6d90)

  


> Demo 地址：https://codepen.io/airen/full/LYMdqdG

  


我想你肯定能创造出更有创意的动画效果。

  


## 小结

  


CSS 动画合成（`animation-composition`）是一个用于控制多个动画在同时影响相同属性时如何组合它们的效果的属性。这对于创建复杂的动画效果非常有用，特别是当你需要控制多个动画之间的相互作用时。

  


`animation-composition` 属性主要有 `replace` 、`add` 和 `accumulate` 。这些值决定了动画效果是替代、叠加还是累积应用到元素属性上：

  


-   `replace` （替代）：使用 `replace` 值时，动画的效果值将完全替代元素的原始值（底层值）。这是默认行为
-   `add` （叠加）：使用 `add` 值时，动画的效果值将与元素属性的原始值（底层值）相加，产生一种叠加效果
-   `accumulate` （累积）：使用 `accumulate` 值时，动画的效果值将与元素属性的原始值进行累积，产生一种累积效果

  


CSS 动画合成对于处理多个影响相同属性的动画非常有用。它允许你根据需要控制动画之间的相互作用，以实现所需动画效果。另外，`animation-composition` 属性通常与多个动画一起使用。动画的应用顺序由 `animation-name` 属性决定，你可以按照 `animation-name` 中列出的顺序应用 `animation-composition` 属性的值 。通过选择合适的 `animation-composition` 值，你可以精确控制多个动画之间的效果，以创建复杂的、精彩的动画效果。

  


总之，CSS 动画合成是一个强大的工具，可用于在多个动画之间控制效果的叠加和组合方式。通过了解如何使用 `animation-composition` 属性，你可以更好地掌握 CSS 动画的创建和控制，实现各种令人惊叹的动画效果。