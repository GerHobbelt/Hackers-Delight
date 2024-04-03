# Approximations of multiplications (on limited hardware)

> **NOTE**: These are my own [@GerHobbelt] notes that are added to my copy of the Hackers' Delight book (2nd ed.) and are *not* part of the book itself. 

This may sound rather loopy/fruity, but say you are running on *severely* limited hardware or suffer from similar extreme time budget constraints and your calculations involve data which can tolerate a decently sized error term, e.g. when processing noisy data or running in a control loop which is tolerant enough of you taking some serious shortcuts to keep the time budget within bounds, then it *might* be an option to replace relatively costly operations in said control loop or other spot you've found yourself currently in.

Of course, assuming very limited hardware, this assumes your CPU

- does not have a hardware multiplier, i.e. the clockcycle count for a single `mul` multiply optcode is prohibitive,
- you're probably also restricted to 8 or 16 bit hardware registers, e.g. when running on a 8051-derivative or other 8 bit processor with its roots "in the classics",

and you *MAY* also have found your shift-left and shift-right operations also cost more than a single cycle, meaning your CPU potentially doesn't come with a barrel shifter built in. Meanwhile, clock cycles have now become more important than *precise* calculations *everywhere*.

You have already converted your relevant C code from floating point (always a costly proposition on small MCUs which don't have IEEE754 hardware built-in) to *fixed point arithmetic* using integer operations.

The clock cycle budget for your calculations are over budget still and assuming you have done everything else, some peephole optimizations *may* save the day, *iff* you can get away with those. 

Which brings us to the multiplications in your operations (**and divisions**, but those are addressed in the other note about approximations!): 

1. multiplications with a constant
2. multiplications with another variable


## multiplications with a constant

These are abundantly covered in the HD book; replacing them with a collection of *shift-left* and *add* operations should do the trick.

As remarked upon in several spots in the HD book, it is beneficial to investigate the *optimal sequence* of these operations, both in terms of *number of operations* and *clock cycle budget*: let's consider one example here, from Chapter 8.4:

> What we want to convey here is that there is more to this subject than meets the eye. First of all, there are other considerations besides simply the number of shift’s and add’s required to do a multiplication by a given constant. To illustrate, below are two plans for multiplying by 45 (binary 101101).
>
> ```
> t = 4x              t1 = 4x
> r = x + t           t2 = 8x
> t = 2t              t3 = 32x
> r += t              r = t1 + x
> t = 4t              t3 += t2
> r += t              r += t3  
> ```
>

The code above was noted as math operations in the book; I use C idiom here for the same.

The book discusses the pros and cons of each of these alternatives, where the variant at the right side may be optimal for a *pipelined*, *multi-register* CPU, e.g. most RISC processors. Meanwhile I find the left variant optimal for simple Z80/8080/8051 family MCUs as those often lack barrel shifters, making the larger power-of-2 multiplications (by left-shifting) costlier, while these processor families also generally lack a set of accumulator-type registers, which would both notably benefit the right-column variant versus the left-side implementation. 

The moral of the story thus far:

If your CPU has multiple accumulators (RISC, etc.) and barrel shifter hardware, i.e. the shift-left opcode is single-tick independent of the shift count, then there's probably more benefit in opcode reordering (done by your compiler, unless you have to resort to writing assembly) and your only way to improve on the "naive approach" is to produce sequences with reduced shift counts.

Producing such short = probably optimal, *minimal* sequences of shift + add operations is a boon on *any* hardware. Also note HD, chapter 8.4 here:

> The methodology described so far is not difficult to work out by hand or to incorporate into an algorithm such as might be used in a compiler; but such an algorithm would not always produce the best code, because further improvement is sometimes possible. This can result from factoring the multiplier `m` or some intermediate quantity along the way of computing `m*x`. For example, consider again `m = 45` (binary 101101). The methods described above require six instructions. Factoring 45 as `5 * 9`, however, gives a four-instruction solution:
>
>       t = 4x + x       // t = 5x
>       r = 8t + t       // r = 9t
>
> Factoring can be combined with the binary decomposition methods. For example, multiplication by 106 (binary 1101010) requires seven instructions by binary decomposition, but writing it as `7 * 15 + 1` leads to a five-instruction solution. For large constants, the smallest number of instructions that accomplish the multiplication may be substantially fewer than the number obtained by the simple binary decomposition methods described. For example, `m = 0xAAAAAAAB` requires 32 instructions by binary decomposition, but writing this value as `2 * 5 * 17 * 257 * 65537 + 1` gives a ten-instruction solution. (Ten instructions is probably not typical of large numbers. The factorization reflects the simple bit pattern of alternate 1’s and 0’s.)
> There does not seem to be a simple formula or procedure that determines the smallest number of shift and add instructions that accomplishes multiplication by a given constant m. A practical search procedure is given in \[Bern\], but it does not always find the minimum. Exhaustive search methods to find the minimum can be devised, but they are quite expensive in either space or time. (See, for example, the tree structure of Figure 15 in \[Knu2, 4.6.3\].)
>

I find that the *factoring approach* above also helps a lot on the most meager hardware, as the individual shift-left operations have smaller shift distance values, which means fewer clock cycles per opcode cost on these hardware platforms, lacking built-in barrel shifters.

Not much more we can do here, unless we agree that our multiplier constants can be *adjusted* to *cheaper* values, i.e. constants which require yet fewer shift operations...


### The need for speed: bearing with approximate calculations

When every opcode counts, it *might* be beneficial to consider *tolerating approximate calculations* in a few places.

To further reduce the time budget required for a multiplication, we can then look at the multiplier `m` and see if we can get away with "adjusting" it to, say, the nearest power-of-2 (implying a single shift-left instead of a bunch of shift-left + add operations) or a further reduction in shift+add ops.

This idea would initially lead us to *adjust* our multipliers to either $m_a = 2^p$ or $m_a = 2^p + 1$ form values, where the latter would cost only a single shift + add op, hence 2 ops and *probably* 2 clock cycles only.

The *error term* would be the difference between our original $m$ and the new $m_a$. From that perspective the $m_a = 2^p + 1$ form is suboptimal as the error term $|m - m_a|$ surely will include higher values than 1, but I find it helps to keep the signal a little "noisy" -- which is *not* a math/calculus consideration and is only useful in particular fringe scenarios.

Meanwhile the obvious way forward here is to reduce the error term as much as possible, which naively means we're only interested in the highest powers-of-2 until we run out of budget. *Be reminded though* about the *factoring* paragraph in DH: the most *efficient* (in terms of error term minimization) approach may be to apply a (partial) factoring before going in and axing a bunch of shift+add ops in there! Take the `m = 45` example, for example: the factored solution takes 2 shifts + 2 adds; suppose we *must* reduce this one and axe one or more ops still, then the more sensible approach would be to axe the add for the 9 factor, as anything else would be worse error-term wise:

- `45 ~ 8 * 5 = 40` --> `t = 4x + x; r = 8t;` (2 shifts, one add) with an error term of 45 - 40 = 5 (`err factor = 5`), vs. doing the same to the other factor:
- `45 ~ 9 * 4 = 36` --> `t = 4x; r = 8t + 1;` (2 shifts, one add) with an error term of 45 - 36 = 9, vs. redoing this as naive, non-prefactored shift + add:
- `45 ~ 32 + 8 = 40` --> `t1 = 32x; t2 = 8x; r = t1 + t2;` (2 (slightly deeper) shifts, one add) with an error term of 45 - 40 = 5, vs. rethinking it *ad hoc* under clock budget restriction conditions (~ 3 opcodes max):
- `45 ~ 32 + 16 = 48` --> `t = 32x; r = 16x + t;` (2 (slightly deeper) shifts, one add) with an error term of 45 - 48 = 3

so you see that the whole 'write as a bunch of shift + add opcodes' is best completely re-evaluated under these clock budget bounds: the best **aproximation** of the multiplier is not necessarily answered by just lopping off the $n$ most significant powers-of-2 in the original multiplier $m$: while I wrote this down I noted that a "little overshooting the mark" here would be beneficial to the error term.

Of course the extra blunt minimum is considering the highest power-of-2 only:

- 45 --> 32 --> error factor = 45 - 32 = 23, while "rounding up" would give:
- 45 --> 64 --> error factor = 45 - 64 = 19, so slightly better for this particular instance,

but eiher way still way worse than the `32 + 16` approximation.

Obviously this must be evaluated on a case by case basis: these tweaks will impact the calculus significantly (and error factor of 3 on a multiplier of 45 still means we'll introduce a $\frac{3}{45} = 6.7\% \approx 7\%$ error in this part of our calculation -- and errors do propagate and have the nasty tendency to *grow* whatever you throw at it, so tread very, very carefully here!


*Anyhoo*, if you haven't attended to your *divisions* in your budget-restricted calculus yet, then caring about your multiplier constants is mere bikeshedding. 
On the way there, we also touch upon the *other* type of multiplications in code...




## multiplications with another variable

DH doesn't spend a lot of time and words on this one for obvious reasons: all larger modern hardware has a hardware multiplier built-in and HD clearly has an obvious 32 bit CPU *bias* as can be observed in the quote below. Section 8.1 is sideways relevant for 8 and 16 bit CPUs without such fancy hardware, though, as there the ubiquitous C `int` can easily involve "multi-word multiply" action *under the hood* when the compiler `int` does not fit in a CPU hardware (accumulator) register:

> This can be done with, basically, the traditional grade-school method. But rather than develop an array of partial products, it is more efficient to add each new row, as it is being computed, into a row that will become the product.
>
> If the multiplicand is `m` words, and the multiplier is `n` words, then the product occupies `m + n` words (or fewer), whether signed or unsigned.
> In applying the grade-school scheme, we would like to treat each 32-bit word as a single digit. This works out well if an instruction that gives the 64-bit product of two 32-bit integers is available. 
>
> Unfortunately, even if the machine has such an instruction, it is not readily accessible from most high-level languages. In fact, many modern RISC machines do not have this instruction in part because it isn’t accessible from high-level languages and thus would not be used often. (Another reason is that the instruction would be one of a very few that give a two-register result.)

Now let's address the scenario where you're riding on top of CPU hardware *sans hardware multiplier*: ah, back to ye olden days!

How to speed up a, say, basic 2-register wide multiply? (16 bit result in a 8 bit CPU, f.e.)

We could take a page from his multi-word multiply book: assuming 8bit hardware like that, we would *not* consider a "half-word" to be an 8-bit `uint8_t` value then as I would still be stuck with an `8*8->8bit mulu`-like opcode, which your hardware does not have: simple, cheap 8 bit MCU, remember?
If we would simply take that `mulmns()` multi-word multiply from the HD book and jack in `uint8_t` as my "half-word type" in there, I'd only make things *worse*, for there's still plenty 8 bit multiplications happening in there and I cannot deliver on those, processor hardware-wise.

One of the choices I see here is to define my "half-word" as being a mere *nibble* (4 bits): in that case I can deliver a "hardware multiplier" that can do even better than `mulu`: `half*half->whole-word mul` by way of a ROM-based lookup table.

Of course, this implies bit-shifting by 4 bits for quite a while, so if your CPU has BCD/nibble-swap opcodes, you're already improving on this.

The other thing to consider is *indexing that lookup table*: clocking at 256 bytes, it'll fit in any minimal hardware, but it might behoof us to ensure our indexing is 8-bit based by folding the `x` and `y` indexes into a single register, turning the two nibbles to be multiplied into a 2-nibble = 1-byte index register value. This, of course, implies that one of the multiplicands must be kept in a 4-bit shifted position, so the processor-specific `mulmns` software implementation will have to reckon with that little nugget as well.


Then there's also the bit about 2-half-word (i.e. single *byte* in our case) multiplications to keep in mind while we look at `mulmns`; say we treat these 'half-words' as our *single digits* in *base* `G` (Dutch: 'grondtal'), then we can write a byte-sized value as

$$v = a_1 * G + a_0$$
	
where $a_0$ is our LSD (Least Significant Digit) and $a_1$ our MSD (Most / Higher Significant Digit) of our byte-sized value $v$.

A multiplication of two such values $v * w$ would then naively look like, suing highschool math:

$$v * w = (a_1 * G + a_0) * (b_1 * G + b_0) = a_1 * b_1 * G^2 + a_0 * b_1 * G + a_1 * b_0 * G + a_0 * b_0$$

which clocks in at 4 multiplications and 3 adds.

The Real OGs before the advent of computer *machines* had a few tricks to reduce the number of operations here (in older times (pre-1960's) this involved a lot of women *computers*) but we're looking at a slightly different scenario from theirs as our cost of overflow of the $a_0 * b_1 * G + a_1 * b_0 * G$ term is bothersome for us: $a_0 * b_1 + a_1 * b_0$ is, worst case, a **9 bit value** as each of these multiplications can potentially produce an 8 bit spanning value. When writing this in assembly language that's not a very big issue as we'll have the carry flag to go with that addition, but it still is a thing to mind when we consider "optimizing" this expression in any way.
It also means we'll need to reckon with 9-bit intermediate register storage if we're not careful. So let's take a look at `mulmns` in HD for starters -- which, for us, would have to be done in nibbles, bytes and double-byte words:

```
// w = u * v
void mulmns(uint8_t w[], uint8_t u[], uint8_t v[], int m, int n) {
   uint16_t k, t, b;
   int i, j;

   for (i = 0; i < m; i++)
      w[i] = 0;

   for (j = 0; j < n; j++) {
      k = 0;
      for (i = 0; i < m; i++) {
#if 0 // original
         t = u[i]*v[j] + w[i + j] + k;
         w[i + j] = t;          // (I.e., t & 0xFF).
         k = t >> 8;
#else // we're only allowing a carry-bit, so 2-bit overflow in t is NOT acceptable!
         t = w[i + j] + k;
         w[i + j] = t;          // (I.e., t & 0xFF).
         k = t >> 8;			// the first carry bit coming out of that addition above
		 t &= 0xFF;				// dummy mask when done in ASM, where t would be an 8-bit register A anyway.
         t += u[i]*v[j];		// can produce the second carry bit
         k += t >> 8;			// the second carry bit
#endif		 
      }
      w[j + m] = k;
   }

   // Now w[] has the unsigned product.  Correct by
   // subtracting v*2**16m if u < 0, and
   // subtracting u*2**16n if v < 0.

   if ((int8_t)u[m - 1] < 0) {
      b = 0;                    // Initialize borrow.
      for (j = 0; j < n; j++) {
         t = w[j + m] - v[j] - b;
         w[j + m] = t;
         b = t >> 15;
      }
   }
   if ((int8_t)v[n - 1] < 0) {
      b = 0;
      for (i = 0; i < m; i++) {
         t = w[i + n] - u[i] - b;
         w[i + n] = t;
         b = t >> 15;
      }
   }
   return;
}
```

This way lies madness as the opcode counts will run amock. However, a few additional thoughts before we leave this one:

- `u[i]*v[j]` is intended to be a nibble-times-nibble-given-single-byte `mulu` style operation: for easy indexing it is useful to have one nibble shifted above the other so the pair form an 8-bit index register value.
- Given that and associativety of the multiply, we *might* turn this into a more suitable byte-times-byte-gives-bytes approach by not shifting these `i` and `j` indexes, but rather masking their nibbles in turn. Alas, we still need the LSN-times-LSN multiplication as well, as well as the MSN-times-MSN, so there's gonna be shifting involved anyway. We just might be able to reduce the number of bit-shift ops in there.



$$
v = (a_1 * G + a_0)

w = (b_1 * G + b_0)

v * w = a_1 * b_1 * G^2 + a_0 * b_1 * G + a_1 * b_0 * G + a_0 * b_0
$$

rewritten with three multiplications, using Karatsuba's approach:

https://en.wikipedia.org/wiki/Karatsuba_algorithm

however this does not bring us much as one of the Karatsuba terms is

z_3 = (x_1 + x_0)(y_1 + y_0)

which implies that this is 5bit by 5bit multiplication, resulting in a lot of potential overflow problems, so we're stuck with the classic Babbage 4 term multiply instead.


Ehhh... Am I wrong? Here ( http://z80-heaven.wikidot.com/advanced-math ) they don't bother about $z_3$ not fitting in a single digit = single byte apparently:

>
> Before we figure out how it works, here is the algorithm:
> 
> ```
> #Want to multiply (ax+b) * (cx+d)
> #ex., in base 10 digits, this could be (a*10+b)*(c*10+d)
> z0 = b*d
> z2 = a*c
> z1 = (a+b)*(c+d)-z0-z2
> result = z0+z1*x+z2*x^2
> ```
> 
> Or another statement of the algorithm (my preferred interpretation):
> 
> ```
> z0 = b*d
> z2 = a*c
> z1 = z0+z2-(a-b)*(c-d)
> result = z0+z1*x+z2*x^2
> ```
> 
> So as an example, we'll multiply 37 by 69.
> 
> ```
> a=3, b=7, c=6, d=9
> z0 = 7*9 = 63
> z2 = 3*6 = 18
> z1 = 63+18-(3-7)*(6-9) = 69   #This is coincidence that z1 is 69 and one of our inputs was also 69 !
> result = 63+690+1800 = 2553
> ```
> 

Wait a minute: he flipped the z3 term. Let's see what the Wikipedia-version does:

```
a=3, b=7, c=6, d=9
z0 = 7*9 = 63
z2 = 3*6 = 18
z1 = (3+7)(6+9) - 7*9 - 3*6  #Here's your overflow in z3 lookin' at ya!
z1 = 150-63-18=150-81=69
```

so, yeah, this checks out the same:

```
result = 63+690+1800 = 2553
```








----




How about... as we remember we'll be facing >= 9-bit intermediate results anyway...

what if we use nibbles on *one side* only, i.e. 8*4 multiplications: a 12-bit index, two bytes per result, an 8K lookup table. Hm. Already sounds much better.
If we then check the multiplicands and swap them so that the smaller one is the one that's going to be *nibble-ized* we've already improved our predicament.

Still, multiplying to 16 bit values on such hardware is no sine cure. Let alone multiplying larger values.




So the next thought is reconsidering rewriting this multiplication as yet another shift+add operation: how bad could it get?

At 16bit x 16bit multiplication, we're looking at bitshifting a (growing from 16 into) 32-bit value 16 times, once for each `1` bit in the second multiplicand. Here we immidiately would consider to swap the multiplicands to place the one with the *least number of 1 bits* at that end: the fewer of these shift operations, the better.

Besides, such hardware generally doesn't come with bigger-than-single-register shift-left operations so a *shift* is not exactly a *single opcode* here either! **Arghhh!**





TBD





Anyway, the base idea here is to do the same as discussed above for multiply-with-constant approximation: find the highest powers-of-2 in the second multiplicand and construct a approximation that takes far fewer cycles while keeping the error term "acceptable".

As you now have a *variable* multiplicand, you cannot hardcode this, so one approach to consider then is to find a *fast* instruction lookup method, which sounds to me like a dispatch table: two base ideas here:

1. take the mutiplicand, shift-left by a few bits (do not take up a lot of ROM space for this!) and index into the ROM code block: fixed space per multiplicand value for a minimal multiplication "function", i.e. jump to the hardcoded multiplier code for this particular byte-suzed multiplicand.

   That'll cost ROM space, though. There's the RTS opcode at the end of each as well, so extra overhead at run-time. **Not smart, I say.**
   
2. Plug the multiplicand in a byte-sized lookup-table, which perhaps specifies two shift operations, so writing each multiply as a shift+shift+add approximation? As we're talking byte-sized multiplicand that's driving this, worst-case shift depth is $n - 1 = 8 - 1$, so we only need 3 bits per shift; as only the most significant shift will need that 7, we have values to spare, e.g. using the shift value '7' in the second slot inside that "instruction" byte to signal there's no second shift required. Heck, we can pack 3 shifts in there as we have 2 bits remaining, so we can improve our approximation by allowing the slightly higher cost of the approxiamtion through 3 shifts + 2 adds.

   We can also improve this idea when it comes to "signal value 7": we stop needing any more shifts when the shift depth value in the current slot is zero(0): *no shift*. We do, however, a signal to identify whether we should add this unshifted value or not. Hmmm..
   When we look at the slots, there's the first slot which will always instruct a shift depth 0..7. The second slot must be able to say "shift by 6 bits", so we need its 3 bits, but that gives us the '7' value in there as signaling value for other things. The last shift slot is only 2 bits, so shift-by-5 is not going to happen, nor is shift-by-4, unless we 'lift' this slot to 3 bits by plugging in an extra 0 bit at the low end: 0..3 --> 2..5, but we need two signal values, not just one, as we must be able to tell the executing code to forego this slot entirely, *or* we accept a minimum error factor of 4 (2^2), which should be quite acceptable for such an approximation scheme anyway, as we are *approximating* the variable multiplication, so exact calculus for certain multiplicand values is a non-feature really: we already accepted a quite high error term to start with, so why then bother with an extra 'signal value' to shut down this third shift op; heck, we might better ask ourselves if we're not better with always executing the same shift,shift,add sequence anyhow and accept that the incoming multiplicand value which just happens to be a power-of-2 just now is also loaded with the error factor like all the others, because we'll add the original value at least: shift value at zero. Reconsidering, it's not much use to add that third shift anyway: the main driver was **speed** here, not *accuracy* and the third shift can only gum up the works and/or complicate the executor, so better not go that deep.
   
   ```
   // multiply u * v:
   //swap(u, v) if u > v  // make v the smallest of the bunch
   //^^^^ how would we know that'll help our error term? It won't have to be that way!
   
   i = instruction_table[v]   // encodes two shift depths, 3 bits each. bits 0..2 + 3..5
   s1 = i >> 3     // first shift depth
   r = u << s1
   s2 = i & 7      // second shift depth
   r += u << s2
   return r
   ```

When splitting a multiply into bit shift ops like that, it's important to have a fast `clz` opcode, at least. See `bsr` opcode on x86, etc.   (C "standard" intrinsic: `__builtin_clz`)

For a kind of worst-case integer multiplication on limited hardware, see the MOS6502 CPU assembly for `_mullong` in SDCC: that CPU is rather limited and multiply is a loop horror shifting (by rotation) across multiple 8-bit values. On that platform at least I surmise the nibble-as-ahalf-word approach above would be beneficial. It takes only 256 bytes extra ROM for the "hardware" `mulu` too.

   
   
   








----

I wonder how the Z80 / 6502 / 8051 run-time libraries do this; hmmm. Check SDCC, HiTech et al...

6502 code is `ror` single bit shifting to span multiple bytes, see `_mulsint`.

Z80 didn't have a multiply opcode either as I recall, either... Yup: https://www.msxcomputermagazine.nl/mccw/92/Multiplication/en.html
They use a special-products approach while using 2 256 byte lookup tables, but theirs only works for inputs up to 64, not the entire byte range we're going for.

https://tutorials.eeems.ca/Z80ASM/part4.htm also has code that does the 16-bit multiplication bit by bit, while its otherwise a riff on the `mulmns` implementation above, just using a 16-bit value for one multiplicand while using a single bit for the other.

https://github.com/z88dk/z88dk/blob/2f5180c28ed5bf858fb23d58f7df235b6cf881be/libsrc/_DEVELOPMENT/math/float/math16/z80/asm_f16_mul.asm#L168 does a fully unrolled 16x16 bit multiply-by-bitshift+add version.


The 8051 has a `mul AB` opcode, so there the story is much easier, e.g. https://pythonhighschool.wordpress.com/2014/02/26/16-bit-multiplication-program-in-8051/ and http://www.8052mcu.com/51mul.phtml : on this MCU, 8x8_16 multiply takes 4 cycles as it carries a hardware multiplier.











References:

- https://github.com/swegener/sdcc
- https://stackoverflow.com/questions/671815/what-is-the-fastest-most-efficient-way-to-find-the-highest-set-bit-msb-in-an-i
- https://github.com/swegener/sdcc/blob/14693f4185f5442d57b380e4210805dcc92d6b0a/src/SDCCicode.c#L2201  (STM8 is indeed listed as not having `mul` opcode; see `_mulsint2slong` assembly code for STM8 though)
- https://github.com/swegener/sdcc/blob/14693f4185f5442d57b380e4210805dcc92d6b0a/device/lib/_fsmul.c#L240
- https://github.com/swegener/sdcc/blob/14693f4185f5442d57b380e4210805dcc92d6b0a/device/lib/stm8/__mulsint2slong.s#L62
- https://wikiti.brandonw.net/index.php?title=Z80_Routines:Math:Multiplication
- https://tutorials.eeems.ca/Z80ASM/part4.htm
- https://www.msxcomputermagazine.nl/mccw/92/Multiplication/en.html
- http://z80-heaven.wikidot.com/advanced-math
- https://en.wikipedia.org/wiki/Multiplication_algorithm#Further_improvements
- https://github.com/z88dk/z88dk/blob/2f5180c28ed5bf858fb23d58f7df235b6cf881be/libsrc/_DEVELOPMENT/math/float/math16/z80/asm_f16_mul.asm#L168
- https://pythonhighschool.wordpress.com/2014/02/26/16-bit-multiplication-program-in-8051/
