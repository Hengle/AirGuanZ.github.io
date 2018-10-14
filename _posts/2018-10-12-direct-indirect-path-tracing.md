---
layout: post
title: 在路径追踪中分别采样直接与非直接光照
key: t20181012
tags:
  - Graphics
---

<!--more-->

## 问题

最近开始写自己的离线渲染器[Atrc](https://github.com/AirGuanZ/Atrc)，撸个暴力的路径追踪器很是容易，代码甚至只有十几行：

{% highlight c++ linenos %}
Spectrum PathTracer::Trace(const Scene &scene, const Ray &r, uint32_t depth) const
{
    if(depth > maxDepth_)
        return SPECTRUM::RED;

    Intersection inct;
    if(!FindClosestIntersection(scene, r, &inct))
        return SPECTRUM::BLACK;

    Spectrum ret;
    if(inct.entity->AsLight())
        ret += inct.entity->AsLight()->Le(inct);

    auto bxdf = inct.entity->GetBxDF(inct);
    auto bxdfSample = bxdf->Sample(-r.direction, BXDF_ALL);
    if(!bxdfSample)
        return ret + bxdf->AmbientRadiance(inct);

    auto newRay = Ray(inct.pos, bxdfSample->dir, 1e-5);
    ret += bxdfSample->coef * Trace(scene, newRay, depth + 1)
         * SS(Abs(Dot(inct.nor, bxdfSample->dir)) / bxdfSample->pdf)
         + bxdf->AmbientRadiance(inct);

    return ret;
}
{% endhighlight %}

这里用一个最大追踪深度来截断过长的路径会导致渲染结果是有偏的，不过这一问题很容易解决（比如引入Russian Roulette策略等），不是本文要讨论的重点。从代码可以看出，路径追踪算法从摄像机镜头出发，沿着一条随机路径不断追寻光源，只有恰好击中了光源的路径才能为最终结果贡献出非零的辐射值（Radiance）。因此，光线越容易击中光源，算法收敛得越快；反之，如果光源很小或是被遮挡得很厉害（需要经过多次反射/折射才能击中），那么一条路径会以很小的概率击中光源并得到一个很亮的点，同时又以极大的概率无法击中光源从而得到黑色，这会导致画面上出现许多噪点。

![PathTracerConvergeTest]({{site.url}}/postpics/Atrc/2018_10_12_PathTracerConvergeTest.png)

图中，中间是场景的几何形体，左边是用淡蓝色的“天空”把场景包裹起来后用100spp渲染的结果，右边则是在场景中添加一个很小的发光球体后用100spp渲染的结果。可以看到，左边的场景中路径很容易击中天空这一无比巨大的光源，而右边的场景中要击中这个小球则是一个概率很小的事件，这导致同为100spp，右边的噪点比左边的明显得多。这并不是场景的明暗导致的，而是光源过小或散射次数过多，很难被采样到导致的（小光源常常意味着较暗的场景，因此有许多人以为噪点多是由于场景亮度低）。

## 理论

回顾路径追踪的理论基础——渲染方程：

$$
L(x \to \Theta) = L_e(x \to \Theta) + L_s(x \to \Theta) = L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega^{\perp}_{\Phi}
$$

其中左侧的$L(x \to \Theta)$表示从$x$处沿方向$\Theta$的出射辐射量，$L_e(x \to \Theta)$是该点作为光源贡献出的辐射，右侧的积分（积分域$\mathcal S^2$是球面立体角）则收集了从各个方向照射到$x$处的光，并计算被反射/折射（$f_s$描述了$x$处物体材质的反射/折射特点）到$\Theta$方向的部分。大部分渲染方面的算法都和这一方程有千丝万缕的联系。

该方程对应的蒙特卡洛估值器$\hat L_N$是：

$$
\hat L_N(x \to \Theta) = L_e(x \to \Theta) + \frac{1}{N}\sum_{i = 0}^N\frac
{f_s(\Phi_i \to x \to \Theta)\hat L_N(x \leftarrow \Phi_i)|N_x \cdot \Phi_i|}
{p(\Phi_i)}
$$

其中对外来光的方向采样了$N$次，采样所使用的概率密度函数为$p$。路径追踪算法所使用的估值器就相当于$\hat L_1$，发射100条路径就等于是将100个$\hat L_1$的估值结果综合起来，这样得到的整体依然是一个$L(x \to \Theta)$的估值器。此外，我们之所以可以在路径追踪中限定路径的最大长度，是因为上式右侧被多次展开后，未被展开部分的系数往往可以小至忽略不计（除非一路上遇到太多的理想镜面，而这种场景极其特殊，在这里不予考虑）。

估值器$\hat L_N(x \to \Theta)$采样的对象是照射到$x$点处的外来光的方向。如果光源很小，那么能够采样到光源的立体角也就很小，而这些方向往往是$L(x \to \Theta)$的主要来源。依照重要性采样的原则，我们应当适当地修改$p$，使得采样到这些方向的概率变得更大。这个方案在实现上比较困难，因此我们考虑另一种方案——将对光源的采样分离出来。由于任意一个$L$都一定可以被分解为$L_e + L_s$两部分，我们将渲染方程右侧的积分用此式稍微展开一下，得到：

$$
\begin{aligned}
L(x \to \Theta) &= L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)(L_e(x \leftarrow \Phi) + L_s(x \leftarrow \Phi))d\omega^{\perp}_{\Phi} \\
&= L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_e(x \leftarrow \Phi)d\omega^{\perp}_\Phi \\
&~~~~~~~~~~~~~~~~~~~~~~~~+\int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_s(x \leftarrow \Phi)d\omega^\perp_\Phi
\end{aligned}
$$

根据立体角微元和面积微元间的关系：

$$
d\omega = \frac{dA^\perp}{r^2}
$$

可以将立体角上的积分转换为场景中所有表面$\mathcal M$上的积分：

$$
\begin{aligned}
&\int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_e(x \leftarrow \Phi)d\omega^\perp_\Phi = \\
&~~~~~~~~~~~~~~~~~~~~~~\int_{\mathcal M}f_s(x' \to x \to \Theta)L_e(x' \to x)V(x, x')
\frac{|N_{x'}\cdot\boldsymbol{e}_{x' \to x}||N_x\cdot\boldsymbol{e}_{x \to x'}|}{|x' - x|^2}dA_{x'}
\end{aligned}
$$

其中$dA$是$\mathcal M$上的面积微元，$V(x', x)$在$x'$与$x$间没有阻碍时为1，否则为0。根据自身性质，$\mathcal M$可以被划分为发光表面$\mathcal M_e$和不发光表面$\mathcal M_n$，其中$\mathcal M_n$上的任意一点$x'$对应的$L_e(x' \to x)$均为0，于是上式的积分域$\mathcal M$可以被缩减为$\mathcal M_e$。这样一来，原本在$\mathcal S^2$上很难采样到小光源对应的方向，现在采样范围变成了光源的表面，哪有采不到的道理？

综上，散射量$L_s$可以被分割为$L_s = E + S$两部分，$E$是光源直接照射到$x$点产生的散射，$S$则是由其他表面散射到$x$点并再次发生的散射。原来的渲染方程现在变成了：

$$
\begin{aligned}
L(x \to \Theta) &= L_e(x \to \Theta) + L_s(x \to \Theta) = L_e(x \to \Theta) + E(x \to \Theta) + S(x \to \Theta) \\
E(x \to \Theta) &= \int_{\mathcal M}f_s(x' \to x \to \Theta)L_e(x' \to x)V(x, x')
\frac{|N_{x'}\cdot\boldsymbol{e}_{x' \to x}||N_x\cdot\boldsymbol{e}_{x \to x'}|}{|x' - x|^2}dA_{x'} \\
S(x \to \Theta) &= \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_s(x \leftarrow \Phi)d\omega^\perp_\Phi
\end{aligned}
$$

对应的估值器也发生了相应的改变：

$$
\begin{aligned}
\hat L(x \to \Theta) &= L_e(x \to \Theta) + \hat E(x \to \Theta) + \hat S(x \to \Theta) \\
\hat E(x \to \Theta) &= \frac 1 {N_E}\sum_{i = 0}^{N_E}\frac
{f_s(x'_i \to x \to \Theta)L_e(x'_i \to x)V(x'_i, x)|N_{x'_i}\cdot\boldsymbol{e}_{x'_i \to x}||N_x\cdot\boldsymbol{e}_{x \to x'_i}|}
{|x'_i - x|^2p(x'_i)} \\
\hat S(x \to \Theta) &= \frac 1 {N_S}\sum_{i = 0}^{N_S}\frac
{f_s(\Phi_i \to x \to \Theta)(\hat E + \hat S)(x \leftarrow \Phi_i)|N_x\cdot\Phi_i|}
{p(\Phi_i)}
\end{aligned}
$$

## 实现

改进后的PathTracer似乎要复杂许多，实际上也就是抄上面的估值器公式而已：

{% highlight c++ linenos %}
Spectrum PathTracerEx::L(const Ray &r, const Intersection &inct, const Scene &scene, int depth) const
{
    const Light *light = inct.entity->AsLight();
    Spectrum Le = light ? light->Le(inct) : SPECTRUM::BLACK;

    auto bxdf = inct.entity->GetBxDF(inct);
    Spectrum Es = E(r, inct, *bxdf, scene, depth);
    Spectrum Ss = S(r, inct, *bxdf, scene, depth);

    return Le + Es + Ss;
}

Spectrum PathTracerEx::E(
    const Ray &r, const Intersection &inct, const BxDF &bxdf, const Scene &scene, int depth) const
{
    Spectrum ret = SPECTRUM::BLACK;
    if(!scene.lightMgr || !lightSampleCount_)
        return ret;

    for(int i = 0; i < lightSampleCount_; ++i)
    {
        auto lightSam = scene.lightMgr->Sample(inct);
        if(!lightSam.light)
            continue;

        auto lightPnt = lightSam.light->SampleTo(inct);
        if(!lightPnt)
            continue;

        lightPnt->pos += 1e-5 * lightPnt->nor;
        Vec3r dstPos = inct.pos + 1e-5 * inct.nor;

        Vec3r dir = dstPos - lightPnt->pos;
        Real dis = dir.Length();

        if(lightSam.light == inct.entity->AsLight() && dis <= 3e-5)
            continue;

        dir /= dis;
        Ray shadowRay = Ray(lightPnt->pos, dir, 0, dis);
        if(Dot(-dir, inct.nor) <= 0.0 || HasIntersection(scene, shadowRay))
            continue;

        ret += bxdf.Eval(-dir, inct.wr) * lightPnt->radiance
             * SS(Abs(Dot(lightPnt->nor, dir) * Dot(inct.nor, -dir))
                / (dis * dis * lightSam.pdf * lightPnt->pdf));
    }

    return ret / lightSampleCount_;
}

Spectrum PathTracerEx::S(
    const Ray &r, const Intersection &inct, const BxDF &bxdf, const Scene &scene, int depth) const
{
    SS RRCoef = 1.0f;
    if(depth >= minDepth_)
    {
        if(depth > maxDepth_ || Rand() > contProb_)
            return SPECTRUM::BLACK;
        RRCoef = 1 / SS(contProb_);
    }

    auto bxdfSample = bxdf.Sample(inct.wr, BXDF_ALL);
    if(!bxdfSample)
        return RRCoef * bxdf.AmbientRadiance(inct);

    auto newRay = Ray(inct.pos, bxdfSample->dir, 1e-5);

    Intersection newInct;
    if(!FindClosestIntersection(scene, newRay, &newInct))
        return SPECTRUM::BLACK;
    auto newBxDF = newInct.entity->GetBxDF(newInct);

    Spectrum EpS = E(newRay, newInct, *newBxDF, scene, depth + 1)
                 + S(newRay, newInct, *newBxDF, scene, depth + 1);

    Spectrum ret = bxdfSample->coef * EpS
                 * SS(Abs(Dot(inct.nor, bxdfSample->dir))
                    / bxdfSample->pdf)
                 + bxdf.AmbientRadiance(inct);
    return RRCoef * ret;
}
{% endhighlight %}