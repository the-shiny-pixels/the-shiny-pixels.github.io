---
layout: post
title:  "Spherical Cone vs Sphere Intersection Test for Spotlight Culling"
tagline: "Tighter spotlight bounds with spherical sectors for better GPU light culling"
description: "A deep dive into using spherical sector vs. sphere intersection tests to optimize spotlight culling in GPU-based rendering, with HLSL implementations and performance tricks."
date:   2025-06-15 00:57:06 +0200
---
I'm sharing here an HLSL implementation for a different flavour of the cone vs. sphere intersection test that we used to improve the culling of spotlights in our Lightmapper.

To accelerate lighting evaluation, our Lightmapper uses a grid structure that subdivides the scene into cubic cells. For each cell, we compute the list of lights whose bounding volume overlaps the cell's bounding box. As no trivial AABB vs. cone exists, a common strategy is to compute instead the intersection test between the cone and the cell's bounding sphere. You can find the code for that approach on [Bart Wronki's blog](https://bartwronski.com/2017/04/13/cull-that-cone).

For a spotlight it's actually possible to compute a tighter bound than a cone, considering that spotlights are usually implemented in rendering engines as point lights with an angular falloff. Effectively, the bounding shape for a spotlight is not a cone but a [spherical sector](https://en.wikipedia.org/wiki/Spherical_sector) (also called spherical cone). That tighter bound formulation is particularly interesting if your spotlight has a wide angle.

![sphere-bounds-as-spherical-cone](/assets/images/conesphere-d.png){:style="display:block; margin-left:auto; margin-right:auto"}
And it turns out it's not too complicated to craft an intersection test between a spherical sector and a sphere. The gist of the idea is to first consider the original sphere from which the spherical cone is carved out and intersect it with the tested sphere. The [sphere-sphere](https://mathworld.wolfram.com/Sphere-SphereIntersection.html) intersection is a circle whose extent is used to define a second spherical sector: 

![sphere-bounds-as-spherical-cone](/assets/images/conesphere-a.png){:style="display:block; margin-left:auto; margin-right:auto"}

It's then just a matter of finding out if the 2 spherical sectors intersect. Given the half angles α and β of the 2 spherical sectors and the angle γ between their 2 axes, they intersect if:
`α + β > γ`

The reasoning holds true as long as the tested sphere is not too close the origin of the original sphere. If it's closer than a certain threshold then the second spherical sector must instead encompass the whole tested sphere.
We can split the second spherical sector's computation into 2 different cases. Let's go through them, the notations used are summarized in the following table.

|Cr| Cone radius |
|Sr| Sphere radius |
|d| Distance from the cone origin to the sphere center |
|α (alpha)| Half-angle of the cone |
|β (beta)| Half-angle of the spherical sector spanned by the sphere |
|γ (gamma)| Angle between the 2 sector's axes |

- **Case A:** The tested sphere is mostly inside the cone's original sphere

![sphere-bounds-as-spherical-cone](/assets/images/conesphere-b.png){:style="display:block; margin-left:auto; margin-right:auto"}
Then, the second spherical sector is tangent to the sphere and its angle is given by: `sin(β) = Sr / d`

- **Case B:** The tested sphere is mostly outside the cone's original sphere 

![sphere-bounds-as-spherical-cone](/assets/images/conesphere-c.png){:style="display:block; margin-left:auto; margin-right:auto"}
Then, from the law of cosines: `cos(β) = (d² - Sr² + Cr²) / (2*Cr*d)`

When the `Cr`, `Sr` and `d` segments form a right triangle, we are sitting inbetween cases A and B. That can be used to determine which formula we should use to compute the spherical sector.  If `d² < Sr² + Cr²`, then Case A formula is used; otherwise, Case B formula is used.

We also need to consider 2 edge cases where the above formulas aren't correct:
- The tested sphere is fully outside the cone's original sphere (when `d >= (Cr + Sr)`), then it's a trivial miss for the intersection test.
- The tested sphere overlaps with the original sphere center (when `d < Sr`), then it's a trivial hit.

To get a better feel of how the test works, you can play around with this [Desmos visualisation](https://www.desmos.com/calculator/3j0xz8ppa4)

Putting now everything together, here is a first naive implementation of our `IntersectSphericalConeWithSphere` test in HLSL:
{% highlight hlsl %}
bool IntersectSphericalConeWithSphere(
    float3 coneOrigin, float3 coneForward, float coneRange, float coneHalfAngle,
    float3 sphereOrigin, float sphereRadius)
{
    const float Cr = coneRange;
    const float Sr = sphereRadius;
    const float Cr2 = Cr * Cr;
    const float Sr2 = Sr * Sr;
    const float3 V = sphereOrigin - coneOrigin;
    const float d2 = dot(V, V);
    const float d = sqrt(d2);

    // Sphere outside of cone's bounding sphere
    if (d2 >= (Cr + Sr) * (Cr + Sr))
        return false;

    // Cone center is inside the sphere
    if (d2 < Sr2)
        return true;

    const float alpha = coneHalfAngle;
    const float beta = (d2 < Sr2 + Cr2) ? asin(Sr / d) : acos((d2 - Sr2 + Cr2) / (2*Cr*d));
    const float gamma = acos(dot(V, coneForward) / d);

    // The spherical sectors intersect if: (α + β) > γ
    return (alpha + beta) > gamma;
}
{% endhighlight %}

`asin` and `acos` are quite expensive to evaluate on the GPU (For a deeper look into their performance impact, see [this article](https://interplayoflight.wordpress.com/2025/01/19/the-hidden-cost-of-shader-instructions/) or [this one](https://seblagarde.wordpress.com/2018/09/03/siggraph-2018-the-road-toward-unified-rendering-with-unitys-high-definition-render-pipeline/)). We can avoid them by massaging a bit the condition:

=> `α + β > γ`

=> `cos(α + β) < cos(γ)` (the comparison is inverted because the cosine function is decreasing on (0,π))

=> `cos(α)cos(β) - sin(α)sin(β) < cos(γ)`

=> `cos(α)cos(β)*d - sin(α)sin(β)*d < cos(γ)*d` (With this last one, we will get rid a few divides by `d`)

Case A and B formulas give us either `cos(β)` or `sin(β)`. The missing `sin(β)` or `cos(β)` value can be derived by using the `cos²(β) + sin²(β) = 1` identity. In its optimized version, the test becomes:
{% highlight hlsl %}
bool IntersectSphericalConeWithSphere(
    float3 coneOrigin, float3 coneForward, float coneRange, float coneHalfAngle,
    float3 sphereOrigin, float sphereRadius)
{
    const float Cr = coneRange;
    const float Sr = sphereRadius;
    const float Cr2 = Cr * Cr;
    const float Sr2 = Sr * Sr;
    const float3 V = sphereOrigin - coneOrigin;
    const float d2 = dot(V, V);

    // Sphere outside of cone's bounding sphere
    if (d2 >= (Cr + Sr) * (Cr + Sr))
        return false;

    // Cone center is inside the sphere
    if (d2 < Sr2)
        return true;

    // Compute the half angle β of the spherical sector that is spanned by the sphere  
    const bool sphereIsClose = (d2 < Sr2 + Cr2);
    const float a = sphereIsClose ? Sr : (d2 - Sr2 + Cr2) * 0.5f * rcp(Cr);
    const float b = sqrt(d2 - a * a);
    const float cosBetaMultD = sphereIsClose ? b : a;
    const float sinBetaMultD = sphereIsClose ? a : b;

    // Angle γ is between the sphere direction and the cone direction
    const float cosGammaMultD = dot(V, coneForward);

    // cos and sin of the cone half angle, precompute if possible
    const float cosAlpha = cos(coneHalfAngle);
    const float sinAlpha = sin(coneHalfAngle);

    // The spherical sectors intersect if: (α + β) > γ
    return (cosAlpha * cosBetaMultD - sinAlpha * sinBetaMultD) < cosGammaMultD;
}
{% endhighlight %}

Note: in a scenario where a single cone is tested against many spheres, `cos(coneHalfAngle)`, `sin(coneHalfAngle)` and `rcp(Cr)` can be precomputed before looping over all the spheres.
