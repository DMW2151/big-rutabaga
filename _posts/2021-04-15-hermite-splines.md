---
layout: default
title:  Oblong is In!
date:   2020-03-24 22:47:53 -0500
permalink: /posts/hermite-splines
---

Last week I learned some new words. Turns out there's a term for the ubiquitous illustration style that features sloping, misproportioned illustrations of people in meetings, driving, biking, climbing rocks, planting flags, piloting rockets, etc. Behold, Corporate Memphis! Looking at all these startup ads got me curious and within 20 minutes I was looking into how to procedurally draw smooth, random, curves.

I learned there are a lot of ways to draw a curve, but in short, you don't want a 27th degree polynomial between dots, nor do you want a child's connect the dot's situation. I found myself reading this article on [cubic hermite splines](https://en.wikipedia.org/wiki/Cubic_Hermite_spline) &mdash; this seems to meet a nice balance, each segment matches it's neighbors segments in location (e.g. `segment_1(x_1)` == `segment_2(x_1)`) and derivative (`d_segment_1(x_1)` = `d_segment_2(x_1)`). That's what I sent out to implement this afternoon.

Reading the attached wikipedia article will give far better context that I can, but the cubic hermite spline uses a combination of 4 basis functions to create a cubic function between any two points. Because any segment in spline-space (my term) can be treated as on [0, 1], As a programmer, we only care about randomizing successive Y values and derivatives.

And for some giggles, let's do it in Julia. Performance is a bit lacking, I'm far from an expert in Julia and I'm sure this could be SIMD'd to the moon. In my implementation randomizing a curve with 100 control points and 100 intermediate points between each pair of points runs in about 2ms. Oof, those allocations are a killer!

```bash
2.360 ms (20881 allocations: 5.29 MiB)
```

![Result](./big-rutabaga/diagrams/result.png)

Cool, so we get some wonky lines. That'll do, solid day's work. I also did a lot of trying to derive the formulas for Nth degree random polynomials where the point, first derivative, and second derivative are the same. Not worth it at all, glad that I tried it, but this was a cool afternoon exercise. Code's below.


```julia
# The Basis Functions...
function h00(t::Float64) return (2t^3) - (3t^2) + 1; end

function h10(t::Float64) return (t^3) - (2t^2) + t; end

function h01(t::Float64) return (-2t^3) + (3t^2); end

function h11(t::Float64) return  (t^3) - (t^2); end

# A'B calcs function value where:
#   A == Array of Basis Funcs
#   B == [position_1, deriv_1, position_2, deriv_2]
function interpolate(p0::Float64, dp0::Float64, p1::Float64, dp1::Float64, n::Int64)
    B = [p0; dp0; p1; dp1]    
    return map(
        x -> [h00(x); h10(x); h01(x); h11(x)]'B, 
        range(0, 1, length=n)
    )
end

# Set Staring Params...
ControlPoints, IntervalLen = 100, 100;
ys, p1, dp1 = [], 0.0, 0.0;

for i in 1:ControlPoints
    p0, dp0, p1, dp1 = p1, dp1, rand(Float64), rand(Float64) # Randomize Point N+1
    _ys = interpolate(p0, dp0, p1, dp1, IntervalLen + 1)

    # No Duplicates on edge points
    ys = vcat(ys, _ys[1:IntervalLen])
end
```

## Reading

* [Splines](https://people.cs.clemson.edu/~dhouse/courses/405/notes/splines.pdf)
* [Hermite Spline Interpolation](https://www.youtube.com/watch?v=p49NFtgEuNs)