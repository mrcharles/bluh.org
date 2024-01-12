---
title: "Code: The Diamond Square Algorithm"
date: 2012-01-31
---
I've been interested in procedural generation in games for a while, so when Notch's entry for Ludum Dare 22 (Minicraft) was released, along with the [source code](http://www.ludumdare.com/compo/ludum-dare-22/?action=preview&amp;uid=398), I went spelunking. I've always been intrigued by Minecraft's world generation, and this gave me a chance to peek behind the curtain at a 2D version (which, in all honestly, is more inline with my sensibilities).

I isolated the core algorithm. It looked vaguely familiar, almost like some of the pixel upsampling algorithms I'd looked at in the past. But I wanted to know if this was something known, or if Notch rolled his own. So I immediately went to my favorite question and answer site: Stack Overflow, where I [asked about it](http://stackoverflow.com/questions/8596125/is-there-a-name-for-this-sampling-algorithm-used-in-minicraft). A quick response pointed me in the right direction.

So now that I knew what it was, I hit up Wikipedia, which [wasn't all that helpful](http://en.wikipedia.org/wiki/Diamond-square_algorithm). I hit up a few related links, and found a bit of random information, but nothing so clear as I would have liked. So I set about rebuilding the code from scratch in my own project to see how it worked.

The first interesting thing I learned from Notch's version, was to modify my point accessor functions to implicitly wrap. This has the side effect of allowing the terrain to wrap perfectly across the edge of the generated texture. Which is great if you are using it to generate a game map!

```csharp
public double sample(int x, int y)
{
  return values[(x &amp; (width - 1)) + (y &amp; (height - 1)) * w];
}

public void setSample(int x, int y, double value)
{
  values[(x &amp; (width - 1)) + (y &amp; (height - 1)) * width] = value;
}
```

These functions let us safely index off the edge of a texture, since it will wrap the data.

Now, we start building our data. Step one is to jump around the texture and initialize a stippled pattern. These are our pure random seed values, which will be interpolated across the texture in order to generate the terrain. The granularity is up to you, but what the spacing accomplishes is a rough control over the 'size' of the various features of the terrain map. If you set your initial step size to the size of your texture, you will usually end up with a large mountain, or basin, depending on how you choose to look at it.

```csharp
double[] values = new double[width * height];

for( int y = 0; y &lt; height; y += featuresize)
  for (int x = 0; x &lt; width; x += featuresize)
  {
    setSample(x, y, frand()); //IMPORTANT: frand() is a random function that returns a value between -1 and 1.
  }
```

You'll note that we actually index off the side of texture; but that is okay since we build our sample functions to wrap.

Now comes the meat of the algorithm. We have two functions we use to set a new sample in the texture.

```csharp
public void sampleSquare(int x, int y, int size, double value)
{
  int hs = size / 2;

  // a   b 
  //
  //  x
  //
  // c   d

  double a = sample(x - hs, y - hs);
  double b = sample(x + hs, y - hs);
  double c = sample(x - hs, y + hs);
  double d = sample(x + hs, y + hs);

  setSample(x, y, ((a + b + c + d) / 4.0) + value);

}

public void sampleDiamond(int x, int y, int size, double value)
{
  int hs = size / 2;

  //  c
  //
  //a x b
  //
  //  d

  double a = sample(x - hs, y);
  double b = sample(x + hs, y);
  double c = sample(x, y - hs);
  double d = sample(x, y + hs);

  setSample(x, y, ((a + b + c + d) / 4.0) + value);
}
```

These two functions do most of the heavy lifting. They take an average of a set of points around the given pixel, and store it back into the data.

Now, the core algorithm.

```csharp
void DiamondSquare(int stepsize, double scale)
{

  int halfstep = stepsize / 2;

  for (int y = halfstep; y &lt; h + halfstep; y += stepsize)
  {
    for (int x = halfstep; x &lt; w + halfstep; x += stepsize)
    {
      sampleSquare(x, y, stepsize, frand() * scale);
    }
  }

  for (int y = 0; y &lt; h; y += stepsize)
  {
    for (int x = 0; x &lt; w; x += stepsize)
    {
      sampleDiamond(x + halfstep, y, stepsize, frand() * scale);
      sampleDiamond(x, y + halfstep, stepsize, frand() * scale);
    }
  }

}
```

So now we go over all the data, and fill in the middle points. We do so by creating new samples with the given pattern, and adding in a random value. I'll get to scale shortly, ignore it for now. It may seem strange at first, but remember, our goal on one pass of the algorithm is to fill in all the midpoints, so that we can run the algorithm again, with successively smaller step sizes, until the texture is filled in completely. We sample the square once, and then do two diamond samples, in order to accomplish this. I've made some really bad programmer art images to highlight this (with apologies to any colorblind readers).

This is our starting state, assuming a 32x32 texture, and an initial feature size of 16. Note that since it's a power of two, we're relying on our earlier wrapping behavior heavily in order to get the desired sampling.

![Image](/img/wp/2012/01/step1.png)

After running the sampleSquare loop, we end up with this:

![Image](/img/wp/2012/01/step2.png)

Numbers have been added to show the order in which they are placed. Remember, each new pixel is sampled with a random variance from the neighbours that form a square. As you can see, we don't have enough data in here to run our diamond square algorithm again. So now we fill in the diamonds:

![Image](/img/wp/2012/01/step3.png)

You should be able to see why we do two diamond samples. The first set gives us the yellow pixels, the second set gives us the green pixels. The numbers in the image are backwards, so the pixels marked 2 are done first, followed by 1. I'd fix the image but I made it on a trackpad and it was hard enough already. Note that this step is relying heavily on the wrapping behavior -- it is wrapping backwards around the texture in order to complete the diamonds.

Now that we have all the individual points filled in, we can now complete our algorithm and build a full terrain texture.

```csharp
int samplesize = featuresize;

double scale = 1.0;

while (samplesize &gt; 1)
{

  DiamondSquare(samplesize, scale);

  samplesize /= 2;
  scale /= 2.0;
}
```

We successively shrink the scale, so that the random variance added to each sample is diminished with step size. The result of this is smooth 'hills' and 'valleys'. We run the algorithm while our sample size is greater than one, and successively shrink that as well, so that at a sample size of 2, we fill in the last step, completing each pixel in the texture.

Below is a completed texture at a size of 128, with a feature size of 16:

![Image](/img/wp/2012/01/done.png)

There you go! You can now generate 2d terrain to your heart's content. I'm just mapping the floating values to a color lerp between black and white, but you can very easily use the value as a z component instead, and make yourself 3d terrain instead!