[home](http://tiny.cc/koff) |
[copyright](https://github.com/koffee/script/blob/master/LICENSE.md) &copy;2017, tim&commat;menzies.us<br>
[<img width=900 src=https://raw.githubusercontent.com/koffee/script/master/img/head.jpg>](http://tiny.cc/koff)<br>
[src](https://github.com/koffee/script/tree/master/src) |
[tour](https://github.com/koffee/script/blob/master/docs/TOUR.md) |
[style](https://github.com/koffee/script/blob/master/docs/STYLE.md)

# NUM

Incremental collector of `lo,hi` and `sd` (standard deviation) of
a stream of numbers. Also, implements some parametric
hypothesis and effect size tests.       

    src   = process.env.PWD + "/../src/" 
    {bsearch,say,want,ninf,inf,max,abs,conf} = require src+'our'
    {Col} = require src+'col'
    {Rand} = require src+'rand'

## Examples

    eg1 = ->
      n = new Num
      (n.add x for x in [9,2,5,4,12,7,8,11,9,3,7,4,12,5,4,10,9,6,9,4])
      say n.mu
      want n.mu==7

    eg2 = ->
      say 1
      n = new Num
      say 2
      n.adds [9,2,5,4,12,7,8,11,9,3,7,4,12,5,4,10,9,6,9,4], (x) -> 0.1*x
      say  "eg1",n.mu,n.sd,n.sd.toFixed(3)
      want n.mu==0.7
      #want n.sd.toFixed(3) == '0.306'

## Code

All the methods marked as `_xxx` extends functionality of the `xxx`
methods defined in the [Col](col.coffee.md) superclass.

    class Num extends Col
      constructor: (args...) ->
        super args...
        [ @mu,@m2,@sd ] = [ 0,0,0,0 ]
        [ @hi, @lo ]    = [ ninf, inf ]

**_add a value to this collector**.
Uses [Welford's incremental sd
algorithm](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Online_algorithm) to update standard deviation.
(thus avoiding the "catastrophic cancellation of precision" seen
with other methods).

      add1: (x) ->
        @lo = if x < @lo then x else @lo
        @hi = if x > @hi then x else @hi
        delta = x - @mu
        @mu += delta / @n
        @m2 += delta * (x - @mu)
        if @n > 1 then @sd = (@m2 / (@n - 1))**0.5

**_normalize a number**  `x` into the range `0..1`, `lo..hi`.

      norm1: (x) ->
        (x - @lo) / (@hi - @lo +  _.tiny)

**Print** contents.

      toString: ->
         " #{@n}:#{@lo}..#{@hi}"

**Sample** from this `Num`'s gaussian space. Requires
a random number generator `r`.

      any: (r)  ->
          w=1
          while w >= 1
            x1= 2.0 * r.next() - 1
            x2= 2.0 * r.next() - 1
            w = x1*x1 + x2*x2
          w = ((-2 * Math.log(w))/w)**0.5
          @mu + @sd* x1 * w

**tTestThreshold** Low-level stuff. Implements look-up table on the
standard t-test critical values table.

      same: (y) ->
        @hedges(y) and @ttest(y)

**Parametric** effect size test:

      hedges: (j,small=0.38) ->
        # https://goo.gl/w62iIL
        nom   = (@n - 1)*@sd**2 + (j.n - 1)*j.sd^2
        denom = (@n - 1)        + (j.n - 1)
        sp    = ( nom / denom )**0.5
        g     = abs(@mu - j.mu) / sp
        c     = 1 - 3.0 / (4*(@n + j.n - 2) - 1)
        #console.log {g0: abs(@mu - j.mu), sp: sp, g: g, c: c}
        g * c < small



      @crit:
        95: {3:3.182, 6:2.447, 12:2.179, 24:2.064, 48:2.011, 96:1.985}
        99: {3:5.841, 6:3.707, 12:3.055, 24:2.797, 48:2.682, 96:2.625}
      ttest1: (x, a=Num.crit[conf],i) ->
        for j,_ of a
          i or= j
          if i <= x <= j
            return a[i] + (a[j] - a[i]) * (x-i)/(j-i)
          if x< i
            return a[i]
          i = j
        a[i]

      ttest: (j) ->
        # true if same
        #  Debugged using https://goo.gl/CRl1Bz
        t  = (@mu - j.mu) / max(10**-64, @sd**2/@n + j.sd**2/j.n)**0.5
        a  = @sd**2/@n
        b  = j.sd**2/j.n
        df = (a + b)**2 / (10**-64 + a**2/(@n-1) + b**2/(j.n - 1))
        abs(t) < @ttest1(df)

Nonparametric effect size test: if the central
tendancy of two lists of numbers are similar,
then return `true`.

    cliffs= (l1,l2, f=((z) -> z) , small=0.147) ->
      """
      with the sort trick on l2, this code handles 100,000
      data points in < 1 sec (whereas otherwise, runtimes
      are > 1 minute)
      """
      l2 = l2.sort((a,b) -> f(a) - f(b))
      m  = l1.length
      n  = l2.length
      lt=gt=0
      for x0 in l1
        x = f(x0)
        i = j = bsearch(l2,x,f)
        while (i< (n-1) and f(l2[i+1]) == x )
          i++
        while (j > 0 and f(l2[j-1]) == x )
          j--
        gt += Math.min(n,n - i)
        lt += Math.max(0,j)
      abs(gt - lt)/ (m*n) < small

Nonparametric bootstrap test: if the distributions
of numbers in two lists are too similar, then
this function returns `true`.

    bootstrap = (y0,z0,conf=0.95,b=1000) ->
       any = (lst) ->
         lst[Math.floor Math.random() * lst.length]
       some = (lst, out=new Num) ->
         (out.add(any(lst)) for _ in lst)
         out
       diff = (y,z) ->
         (y.mu - z.mu) / (10**-64 + (y.sd/y.n + z.sd/z.n)**0.5)
       #------------------------
       [x,y,z] = [new Num, new Num, new Num]
       (y.add y1 for y1 in y0)
       (z.add z1 for z1 in z0)
       (x.add y1 for y1 in y0)
       (x.add z1 for z1 in z0)
       t    = diff(y,z)
       yhat = (y1 - y.mu + x.mu for y1 in y0)
       zhat = (z1 - z.mu + x.mu for z1 in z0)
       more = 0
       for [1..b]
         more++ if diff(some(yhat), some(zhat))  > t
       (more / b) <  conf

Check if two lists of numbers come from the same distribution
(using a non-parameteric effect size test, then a bootstrap).

    same = (l1,l2) ->  
      cliffs(l1,l2) and bootstrap(l1,l2)

Utility: print some percentiles out of a list of numbers

    tiles = (lst,want=[25,50,75]) ->
      lst = lst.sort ((a,b) -> a - b)
      pos = (n) -> Math.floor(n*0.01* lst.length ) 
      (lst[ pos(n) ] for n in want)
     
Utility: preppty print some percentiles, formated to `f` decimal
places.

    nums = (lst, want, f=3) ->
      (n.toFixed(3) for n in tiles(lst,want,f)).toString()


## End stuff

    @Num = Num
    @tests=[eg1,eg2]
    @cliffs = cliffs
    @bootstrap=bootstrap
    @nums=nums
    @same = same
   
