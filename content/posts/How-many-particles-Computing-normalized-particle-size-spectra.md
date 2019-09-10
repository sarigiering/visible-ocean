+++
title = "How many particles? Computing normalized particle size spectra"
date = 2019-08-22T13:55:12-02:30
draft = false
tags = ["particles", "size spectrum", "FlowCam"]
categories = []
+++

Using size spectra to describe the ocean ecosystem is the new craze, particularly as _in situ_ imaging devices are becoming cheaper and more capable. The theory behind it is simple: Small particles, such as bacteria and phytoplankton, are numerous, while large particles, such as whales, are sparse. As a consequence, the summed biomass of small organisms far outweighs the summed biomass of large organisms. We refer to this trend as the size spectrum, and it has a negative slope.

The interpretation of the slope (and deviations from its linear nature) have been used to infer food-web structure, predation, population growth, and more. Using a simple mathematical approach (e.g. a linear regression between binned biomass and particle size) is convenient as it saves the observational biologist a lot of time in front of the microscope, and modellers can happily and easily apply allometric equations to the data.

Life is obviously not that simple. But let's for one moment believe it is.

## Just how many particles do I need to count?
Let's get to the practical question. When it comes to analyzing a sample with the aim to calculate a normalized particle size spectrum, it is important that you count sufficient particles in each bin to calculate the slope with certainty. If there are insufficient particles in the bins, the size spectrum becomes 'unstable' and an outlier can have a dramatic effect on the data interpretation.

Fortunately, you can decide on a rough number of particles before you start.


For the **normalized abundance spectrum** that we will calculate, we found that

* a good starting point is **10-20 particles** in your largest bin. 
* you want **at least 4 solid bins** (i.e. bin containing >10 particles).
* we like the octave scale (using **log2**) (I'll post about different log-scales another time). 

## Calculate minimum number of particles needed for size spectra

First I build a data frame that contains the basic information for composing a size spectra:

+ _L_ = lower limit of size for each bin
+ _w_ = log2 of lower size bin
+ &Delta;_w_ =  width of size bin (in &mu;m)
+ w&#772; = mean of size bin<sup>[1]</sup>

Now let's write this in R.

```R
dat   <- data.frame("L" = 2^c(0:12))
dat$w <- log2(dat$L)
for(i in 1:nrow(dat)) dat$w_avg[i] <- (dat$w[i]+dat$w[i+1])/2
for(i in 1:nrow(dat)) dat$dw[i]    <- dat$w[i+1]-dat$w[i]

print(dat)
```

bin | minimum length (_L_ in &mu;m)   | w (log<sub>2</sub> of _L_) | bin mean (w&#772;) | bin width (&Delta;_w_)
:---:|:---:|:---:|:---:|:---:
1   | 1   | 0   | 0.5   | 1
2   | 2   | 1   | 1.5   | 1
3   | 4   | 2   | 2.5   | 1
4   | 8   | 3   | 3.5   | 1
5   | 16  | 4   | 4.5   | 1
6   | 32  | 5   | 5.5   | 1
7   | 64  | 6   | 6.5   | 1
8   | 128 | 7   | 7.5   | 1
9   | 256 | 8   | 8.5   | 1
10  | 512 | 9   | 9.5   | 1
11  | 1024| 10  | 10.5  | 1
12  | 2048| 11  | 11.5  | 1
13  | 4096| 12  |       | 
       
I then calculate the intercept *a* for a superspectrum (Blanco _et al._ 1994), where the number of particles (*n*) in our largest size bin is 20.

![alt text](https://drive.google.com/uc?id=1VaYNniMcEc5kbNguFlDlLbl-DI6Uy1PF)

When looking at this graph, beware of the following two things:

1. Blanco _et al._ (1994) use "N" for "number of observations". This annotation confuses me. For my codes, I therefore use _n_ rather than _N_. 
1. The x-axis shows our parameter w&#772;

As this is a really nice linear regression, we get a simple linear equation:

log(_n_/&Delta;V) = _a_+_b_*log(V)  

As I am using the width (equivalent spherical diameter) here instead of volume (_V_), my equation looks like this:

log(_n_/&Delta;_w_) = _a_+_b_*w&#772;


---

### Caution: Normalized vs. standard number size spectrum
We now want to calculate the number of particles in a certain size bin. So what we really need here is the 'number size spectrum', which is not normalized to the width of the size bin. To 'denormalize', we multiply by the bin width scaled to the mean bin size:

log(_n_) = _a_ + _b_*w&#772; + &Delta;_w_*w&#772;

log(_n_) = _a_ + (_b_+&Delta;_w_)*w&#772;


This equation is pretty cool as it shows how the slope of the normalized size spectra (_b<sub>norm</sub>_) and 'standard' size spectra (_b<sub>standard</sub>_) relate to each other: 

_b<sub>norm</sub>_ = _b_ 

_b<sub>standard</sub>_ = (_b_+&Delta;_w_)

And as we have set up our &Delta;_w_ to be 1, we get:

(_b<sub>norm</sub>_) = (_b<sub>standard</sub>_)+1.

Sweet!

---
Now that we have that sorted, we can almost start calculating numbers. I use the slope of Blanco _et al_'s (1994) normalized superspectrum and assume that a size bin is well represented if there are 20 particles in it.

So we assume:

* _b_ = -2 
* _n_ = 20 

The last parameter that we need is the intercept. We can calculate it by rearranging the equations as:

_a_ = log(_n_) - (b+&Delta;_w_)*w&#772;

_a_ = log(20) - (-2+1)*w&#772;


```R
b <- -2
n <- 20

dat[,"a_when_n=10"] = log2(n) - (b+dat$dw)*dat$w_avg
```

Now that I know the intercept, I can compose a number size spectrum with varying population sizes. In the following table, you see a column for each of these spectra. For example, the column *n_w=0* shows a number size spectrum where the number of particles in the size bin starting at log<sub>2</sub>(_L_)=0 has 10 particles. 


```R
for(i in 1:(nrow(dat)-1)){
  dat[,paste("n_w=",dat$w[i],sep="")] <- 2^(dat[,"a_when_n=10"][i]+ dat$w_avg*b + dat$dw*dat$w_avg)
  }
  
print(round(dat))
```

          L  w w_avg dw a_when_n=10    n_w=0 n_w=1 n_w=2 n_w=3 n_w=4 n_w=5 n_w=6 n_w=7 n_w=8 n_w=9 n_w=10 n_w=11
    1     1  0     0  1         4.8       20    40    80   160   320   640  1280  2560  5120 10240  20480  40960
    2     2  1     2  1         5.8       10    20    40    80   160   320   640  1280  2560  5120  10240  20480
    3     4  2     2  1         6.8        5    10    20    40    80   160   320   640  1280  2560   5120  10240
    4     8  3     4  1         7.8        3     5    10    20    40    80   160   320   640  1280   2560   5120
    5    16  4     4  1         8.8        1     3     5    10    20    40    80   160   320   640   1280   2560
    6    32  5     6  1         9.8        1     1     3     5    10    20    40    80   160   320    640   1280
    7    64  6     6  1        10.8        0     1     1     3     5    10    20    40    80   160    320    640
    8   128  7     8  1        11.8        0     0     1     1     2     5    10    20    40    80    160    320
    9   256  8     8  1        12.8        0     0     0     1     1     2     5    10    20    40     80    160
    10  512  9    10  1        13.8        0     0     0     0     1     1     2     5    10    20     40     80
    11 1024 10    10  1        14.8        0     0     0     0     0     1     1     2     5    10     20     40
    12 2048 11    12  1        15.8        0     0     0     0     0     0     1     1     2     5     10     20

So if you wanted to count all particles from 4 mm in diameter down to our smallest size bin here (1-2 &mu;m), you would expect to count 81,900 particles. That's a very high number and, most likely, your device will not be able to cover such a wide range of sizes. So how can you use this table?

## Interpret and apply this table

You can see that the particle numbers pretty quickly become high. It is unlikely that you want to or can count ten's of thousands of particles owing to logistical constraints (time, sample volume, _etc_). Moreover, your device will have a limited range where it can count particles with sufficient accuracy. 

In our case, we are using the FlowCam and have 4 magnifications to choose from:

 Objective | Min size (&mu;m) | Max size (&mu;m) | Bin range
      :---:|             :---:|             :---:|    :---:
      20x  |    1             |    50            |  1 to 5
      10x  |    2             |   100            |  2 to 6
       4x  |    5             |   300            |  4 to 8
       2x  |    12            |  1000            |  4 to 10

Let's say we want to use magnifications 20x and 4x. We want to make sure that we have a reasonably high number of particles in our largest bin. For a 20x magnification, the largest bin I can realistically capture completely ranges from 16 to 32 &mu;m. If I count 1260 particles, I can expect to find around 40 particles in this bin if the slope of the normalized number spectrum is indeed -2. If the slope was steeper, the number of particles in this bin would get smaller, nearing our lower accepted level of 10-20 particles per bin. So, 1260 particles is a good minimum number for my 20x magnification as I have a bit of 'air' in my bin counts to accommodate for a steeper slope. 

```R
smallest_bin <- 1
largest_bin  <- 5
name         <- paste("n_w=",largest_bin, sep="")
range        <- smallest_bin:(largest_bin+1)

print("--- 20x magnification ----")
print(dat[range,c("L","w",name)])

print(paste("Total:", round(sum(dat[range,c(name)], na.rm=T))))
```

    [1] "--- 20x magnification ----"
       L w n_w=5
    1  1 0   640
    2  2 1   320
    3  4 2   160
    4  8 3    80
    5 16 4    40
    6 32 5    20
    [1] "Total: 1260"

```R
smallest_bin <- 4
largest_bin  <- 8
name         <- paste("n_w=",largest_bin, sep="")
range        <- smallest_bin:(largest_bin+1)

print("--- 4x magnification ----")
print(dat[range,c("L","w",name)])

print(paste("Total:", round(sum(dat[range,c(name)], na.rm=T))))
```

    [1] "--- 4x magnification ----"
        L w n_w=8
    4   8 3   640
    5  16 4   320
    6  32 5   160
    7  64 6    80
    8 128 7    40
    9 256 8    20
    [1] "Total: 1260"

## Footnotes:
<sup>[1]</sup> Here I use the arithmetic mean. Later, when computing the actual size spectra, you may want to use the geometric mean.

## References:

Blanco JM, F Echevarría, CM García (1994). Dealing with size-spectra: some conceptual and mathematical problems. Sci Mar 58:17-29

Álvarez E, Á López-Urrutia, E Nogueira, S Fraga (2011). How to effectively sample the plankton size spectrum? A case study using FlowCAM. J Plankton Res 33(7):1119–1133, https://doi.org/10.1093/plankt/fbr012

Sprules W, LE Barth (2016). Surfing the biomass size spectrum: some remarks on history, theory, and application. Can J Fish Aquat Sci 73:477-495. https://doi.org/10.1139/cjfas-2015-0115