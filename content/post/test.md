---
title: "Test"
date: 2021-01-02T17:59:23-05:00
draft: true
# markup: md
---

# Arknights-Pull-Simulation

Estimate the theoretical probability of getting a **specific 6★** operator **on $i^{th}$ pull/within $i$ pulls** in different types of banners in Arknights via Monte Carlo method.

The starting point of the pity system is configurable. I provided this flexibility for you if you are curious about the probability in a different mathematical model. With different settings for this value, you will see that the pity system has a different influence on the theoretical probability.

In this README, I will introduce how to use this code, giving several simple examples than you can have a first try. I will also discuss the mathematical model behind the pulling rules in Arknights, as well as the probabilities of several kinds of random events that you may care about. Then the key part of the code — random number generator — will be briefly explained, which I believe is inspiring and important. Finally, the estimated theoretical probability will be presented and analyzed, and have a look at some future work.

You can also jump to the part that you interested in directly:

- [What is Arknights?](#what-is-arknights-)
- [Usage](#usage)
    + [Build the Code](#build-the-code)
    + [Command Line Arguments](#command-line-arguments)
    + [Some Further Explanation](#some-further-explanation)
    + [Quick Examples](#quick-examples)
- [The Mathematical Model In Arknights](#the-mathematical-model-in-arknights)
    + [The Arknights Pulling Rules and Headhunting Banners](#the-arknights-pulling-rules-and-headhunting-banners)
    + [Abstracted Questions and Definitions](#abstracted-questions-and-definitions)
    + [6★ Operator Drop: the Probability and the Expectation](#6★-operator-drop:-the-probability-and-the-expectation)
    + [Target 6★ Operator Drop: Attempts and Approaches of Calculating](#target-6★-operator-drop:-attempts-and-approaches-of-calculating)
      - [Promising Theoretical Approaches](#promising-theoretical-approaches)
      - [This Repo's Approach: Monte Carlo Simulation](#this-repo-s-approach--monte-carlo-simulation)
- [Generating the Random Numbers](#generating-the-random-numbers)
    + [Pitfall: `rand() % n` is Evil!](#pitfall---rand-----n--is-evil-)
    + [Favored Approach - C++11 `<random>` Library](#favored-approach---c--11---random---library)
- [Result Analysis](#result-analysis)
    + [Estimated Probability of Double-Rate-Up Limited Banner](#estimated-probability-of-double-rate-up-limited-banner)
    + [Estimated Probability of Double-Rate-Up Standard Banner](#estimated-probability-of-double-rate-up-standard-banner)
    + [Estimated Probability of Single-Rate-Up Standard Banner](#estimated-probability-of-single-rate-up-standard-banner)
    + [Estimated Probability of Single-Rate-Up Limited Banner](#estimated-probability-of-single-rate-up-limited-banner)
    + [Comparison Across Different Types of Banners](#comparison-across-different-types-of-banners)
    + [Summary](#summary)
    + [Error Analysis](#error-analysis)
- [Related Works](#related-works)



# What is Arknights?

![](img\Arknights Introducing Banner.png)

**Arknights** is a [tower defense](https://en.wikipedia.org/wiki/Tower_defense) mobile game developed by Chinese developers Studio Montagne (蒙塔山工作室) and [Hypergryph](https://www.hypergryph.com/#/). It was released in China on 1 May 2019, and worldwide on 16 January 2020 by [Yostar](https://yo-star.com/pc/index.html). Arknights is available on iOS, Android and PC platforms and features [gacha game](https://en.wikipedia.org/wiki/Gacha_game) mechanics. (Adopted From *Wikipedia*)

The game has hit No.1 overall on the iOS App Store top grossing ranking list for three times [[1](https://weibo.com/arknights?sudaref=www.google.com&ssl_rnd=1608151703.5481&is_hot=1)], TapTap Game Awards 2019 - Best Game, TapTap Game Awards 2019 - Most Influential China-Developed Game, TapTap Game Awards 2019 - Gamers' Choice [[2]](https://weibo.com/arknights?profile_ftype=1&is_all=1&is_search=1&key_word=%E7%95%85%E9%94%80%E6%A6%9C#_0), Bilibili 2019 Game Review - Top Rated Game, Bilibili 2019 Game Review - Most Searched Game, Bilibili 2019 Game Review - Users' Favorite Mobile Game [[3]](https://www.bilibili.com/blackboard/activity-gamereview2019.html), etc.

The game has achieved 40 million downloads worldwide [[4]](https://www.sonymusic.co.jp/artist/ReoNa/info/518163) and 7 millions download outside the CN Server at the end of 2020, April. [[5]](https://twitter.com/ArknightsEN/status/1255542361632813058)

Official Website:

* [Arknights Global](https://arknights.global/)
* [HyperGryph](https://www.hypergryph.com/#/)

# Usage

### Build the Code

After `git clone`, `cd` into the directory and run `make` in the repo's directory to build from the source code. Then an executable file named `simulation_sequential` will be generated.

Run `make clean` to remove all `*.o`s and the executable files.

### Command Line Arguments

You need to specify some command line arguments to run the simulation:

```shell
./simulation_sequential [--help] [-t|--total-pull-time <value>] [--standard|--limited] 
                        [-p|--pity <value>] [-n|--num-rate-up <value>] 
```

The arguments in square brackets are optional, and their orders does not matter. You can use either the short name arguments (`-t`, `-p` and `-n`) or the long name arguments (`--total-pull-time`, `--pity` and `--num-rate-up`) as you wish. 

Each arguments' meaning is explained in the following table:

| Argument                     | Explanation                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| `--help`                     | Show the help message - the introduction of each argument and how to run the program <br/>Should not be specified with other arguments at the same time |
| `-t`<br/>`--total-pull-time` | Set how many times of pulling will be executed during the simulation<br/>Control the scale of the simulation<br/>**Valid value: an integer between $[1, 18446744073709551615]$ (inclusive)** |
| `--standard`                 | Estimate the theoretical probability in a standard banner    |
| `--limited`                  | Estimate the theoretical probability in a limited banner     |
| `-p`<br/>`--pity`            | Set the starting point where the pity system comes into effect<br/>The pity system will start to increase the probability of getting a 6★ operator in the pull after the `N-th` pull (`N` is the number you specified)<br/>**Valid value: an integer from $[0, 4294967295]$ (inclusive)** |
| `-n`<br/>`--num-rate-up`     | Set whether to simulate a single-rate-up banner or a double-rate-up banner<br/>**Valid value: either $1$ or $2$** |

On default, if no arguments are provided, the program will simulate a double-rate-up limited banner with $100,000,000$ times pulling with pity starting point equals to $50$ (equivalent to run with `--limited -t 100000000 -n 2 -p 50`), which is same as in a limited banner in Arknights, unless you specify the corresponding arguments to override the default behavior of the program.

### Some Further Explanation

* The bigger the value for total pull time is, the more accurate the result will be, but the program will consume more time to finish the simulation. The single thread simulation with `-t 10000000000` (simulating $10$ billion times of pulling) took about $110$ seconds to finish on the platform that I used:

  * OS: Ubuntu 18.04
  * CPU: Intel® Xeon® Gold 6140 CPU @ 2.30GHz
  * Memory: 8GB

* You can override the program's default simulation configuration. For example, if you only specify `-t 200000000` (or `--total-pull-time 200000000`) and still omit other arguments, the default value of total pull times (which is $100,000,000$) will be overridden, and the theoretical probability of a double-rate-up limited banner will be estimated under $200,000,000$ times simulated pulling. If you also specify `--standard` and `-n 1`, then the program will simulate a single-rate-up standard banner under $200,000,000$ times simulated pulling.

* The program will try its best to find out any types of syntax errors that may occur in the command line arguments and give you the error reason (show in the picture below) — just like a simple compiler. However, do not fully rely on this error-check feature: always read the docs carefully ;-)

  <img src="img\Arguments Error Checking.png" style="zoom:33%;" />

### Quick Examples

Here are several quick examples to start with:

* See the help message

  `./simulation_sequential --help`

* Simulate a double-rate-up limited banner

  `./simulation_sequential --limited -t <a-positive-big-integer>`

  or just

  `./simulation_sequential -t <a-positive-big-integer>`

* Simulate a double-rate-up standard banner

  `./simulation_sequential --standard -t <a-positive-big-integer>` 

* Simulate a single-rate-up standard banner

  `./simulation_sequential --standard -n 1 -t <a-positive-big-integer>`

* Simulate a single-rate-up limited banner (not appeared in Arknights yet!)

  `./simulation_sequential --limited -n 1 -t <a-positive-big-integer>`

  or just

  `./simulation_sequential -n 1 -t <a-positive-big-integer>`
  
  The recommended value for `<a-positive-big-integer>` is $100000000$ or above.
  
  It is recommended that you redirect the output to a file, for example
  
  ```shell
  ./simulation_sequential --limited -t 200000000 >> simu_limited_banner.res
  ```
  
  since the output of the simulation program will be relatively long.

# The Mathematical Model In Arknights

In this part, I will introduce the mathematical model behind the Arknights pulling rules, the definitions of random events, which are important to understanding the simulation results, and several method to calculate those probabilities.

### The Arknights Pulling Rules and Headhunting Banners

In Arknights, the probability of getting a 6★ operator in a pull is $2\%$. However, there also exists a rule called the **Pity System** — if you have not get any 6★ operator after consecutive $50$ pulls, this probability will increase by $2\%$ in each following pull — until you get a 6★ operator and reset this probability to $2\%$ as well as the counter of the pity system, which counts how many times did you pull without getting a 6★ operator.

Detailly, suppose some guy did not get any 6★ in last $50$ pulls, on his $51^{st}$ pull, the probability will be $4\%$; and if he still did not get a 6★ operator on his $51^{st}$ pull, then on his $52^{nd}$ pull, this probability will be $6\%$, and so on...

This mechanism ensures that even the most unlucky player can still obtain a 6★ operator in his/her $99^{th}$ pull.

> The probability of this tragedy — $98$ pulls without getting a 6★ operator — theoretically, is
> 
> $$
> \begin{aligned}
> Pr(Uninstalling\ Arknights) =& (1-2\\%) ^ {50} \times (\prod_{i=51}^{98}(1-(2\% \times (i-50)+2\%))) \times 1\\
> &= (0.98)^{50} \times \prod_{i=1}^{48} \frac{49-i}{50}\\
> \approx & 1.27248 \times 10^{-21}
> \end{aligned}
> $$
>
> which is much, much, much less than being hit by a lightning, which is approximately $2 \times 10^{-6}$, according to the data from Centers for Disease Control and Prevention [[6]](https://www.cdc.gov/disasters/lightning/victimdata.html)... poor guy...

Operators are obtained in headhunting banners. There are basically two general kinds of headhunting banners in Arknights,

1. Standard Banner (Standard Headhunting) — banner without limited 6★ operator (e.g., Nian). This kind of banner can either has one rate-up 6★ operator, which often appears when new events are released, or has two rate-up 6★ operators.

   For example, *Lisa of the Valley*, *Blemishine (CN Server)*, *Standard Pool #26* are standard banners.
   
   <p>
   	<img style="margin-left: auto; margin-right: auto;" src="img\standard-banner-lisa-of-the-valley.png" width="300" />
   	<img style="margin-left: auto; margin-right: auto;" src="img\standard-banner-blemishine.jpg" width="300" />
   	<img style="margin-left: auto; margin-right: auto;" src="img\standard-banner-026.png" width="300" />
   </p>
   <pre><center style="font-size:14px;color:#C0C0C0;">  Lisa of the Valley                      Blemishine                   Standard Headhunting #26 </center></pre>
   
2. Limited Banner (Limited Headhunting) — banner with one limited 6★ operator. Currently this kind of banner has two rate-up 6★ operators.

   For example, *Earthborn Metals*, *Cremation Last Wish (CN Server)*, *Forget Me Not (CN Server)* are limited banners.
   
   <p>
   	<img style="margin-left: auto; margin-right: auto;" src="img\limited-banner-earthborn-metals.png" width="300" />
   	<img style="margin-left: auto; margin-right: auto;" src="img\limited-banner-cremation-last-wish.jpg" width="300" />
   	<img style="margin-left: auto; margin-right: auto;" src="img\limited-banner-forget-me-not.jpg" width="300" />
   </p>
   <pre><center style="font-size:14px;color:#C0C0C0;">  Earthborn Metals                   Cremation Last Wish                   Forget Me Not        </center></pre>

> The rolls of not obtaining 6★ operator will be accumulated in all the standard headhunting, and they will not be reset when a standard headhunting ends. The increased odds caused in one event also applies to another standard headhunting. (The description in Arknights)

> There also exists a type of banner called Joint Operation. The Joint Operation banner has a different rule. Currently this simulation program does not support this type of banner, but I will soon add this functionality.

In a standard banner, if you get a 6★ operator, there is a $50\%$ chance that the said operator will be the rate-up one(s); in a limited banner, if you get a 6★ operator, there is a $70\%$ chance that the said operator will be the rate-up one(s). Suppose that **your target is one of the featured 6★ operators in a banner**, the chances can be expressed as a conditional probability

$$
\begin{aligned}
&Pr(Get\ the\ Target\ Star\ 6\ Operator\ | \ Get\ a\ Star\ 6\ Operator\ in\ Standard\ Banner) = 50\% \\
&Pr(Get\ the\ Target\ Star\ 6\ Operator\ | \ Get\ a\ Star\ 6\ Operator\ in\ Limited\ Banner) = 70\%
\end{aligned}
$$

**Here, we made an important assumption - if $n$ operator(s) are featured in a banner (i.e., rate-up), they equally share this conditional probability.** That is, the chance that the obtained 6★ operator in a banner is one of the featured operators equals to $50\% / n$ or $70\% / n$. This looks reasonable enough, although the Arknights official has not explicitly claimed this.

What's more, Arknights has a secondary pity system called **Spark System** for limited banners. You can get one *Recruitment Data Contract* after each pull in the limited banner. And the 6★ operator featured in this limited banner can be bought in the store using 300 *Recruitment Data Contract*. ~~You will not want to use it...~~

### Abstracted Questions and Definitions

Here, we define the following random events,

* $A_i$ : Get a 6★ operator **on** your $i^{th}$ pull
* $B_i$ : Did not get any 6★ operators **within** $i$ pulls
* $S_i$ : Get the **target** 6★ operator **on** your $i^{th}$ pull
* $W_i$ : Get the **target** 6★ operator **within** $i$ pulls
* $F_i$ : Did not get the **target** 6★ operators **within** $i$ pulls

Here the **target 6★ operator** refers to the (one of the) 6★ operator(s) featured by the banner, and the $A_i$ and $S_i$ require that **6★/target 6★ operator is not obtained in all previous $i-1$ pulls**.

To make the notation simpler,

* The subscript $i$ in $A_i$ starts counting from $i=1$, until you get a 6★ operator (and then counting from 1 again). Based on the rules of pity system, the maximum value for $i$ is $99$.
* The subscript $i$ in $B_i$ starts from $i=0$ on the pull in which you get a 6★ operator, and every time you did not get a 6★ operator in a pull, $i$ increased by $1$. Based on rules of the pity system, the maximum value for $i$ is $98$.
* The subscript $i$ in $S_i$ and $W_i$ starts counting from $i=1$, until you get the **target** 6★ operator (and then counting from 1 again). Since there always exists a chance of getting a non-target 6★ operator, $i$ can be $+ \infty$.
* The subscript in $F_i$ starts from $i=0$ on the pull in which you get the **target** 6★ operator, and every time you did not get the **target** 6★ operator in a pull, $i$ increases by $1$. Note that $i$ will not start from $0$ again if you get a non-target 6★ operator in a pull. Since there always exists a chance of getting a non-target 6★ operator, $i$ can be $+ \infty$.

In one word, when you achieve your goal, $i$ resets and starts counting from the beginning again.

Based on the definitions above, here we give several examples,

* $A_{42}$ means that since your last time of getting a 6★ operator, you did not get a 6★ operator in your following $41$ pulls, and you finally get one on your $42^{nd}$ pull. This is same as both $B_{41}$ and $A_{42}$ happen.
* $B_{37}$ means that, suppose you get a 6★ operator on your previous pull, in the following continuous $37$ pulls, you do not get any 6★ operator. $B_0$ also means $A_1$.
* $S_{54}$ means that since your last time of getting the **target** 6★ operator, you did not get **the same operator** in your following $53$ pulls, and finally get this operator on your $54^{th}$ pull. This is same as both $F_{53}$ and $S_{54}$ happen.
* $W_{25}$ means that since your last time of getting the **target** 6★ operator, you get the **target** 6★ operator **within** $25$ pulls. Note that within these $25$ pulls, whenever you get the operator, you just stop pulling.
* $F_{17}$ means that, suppose you get the **target** 6★ operator on your last pull, in the following continuous $17$ pulls, you do not get the **target** 6★ operator. $F_0$ also means $S_1$.

Who is the target 6★ operator is determined by you, as long as this operator is featured by a banner.

> Please forgive me being wordy here. (I am not a native speaker of English) I hope I made myself clear by giving these examples that explain my definitions above :-)

**We are curious about the values of $Pr(S_i)$ and $Pr(W_i)$, which are also what this repo attempts to estimate.**

### 6★ Operator Drop: the Probability and the Expectation

Calculating the probability that getting a (any) 6★ operator is relatively easy. Some previous works [[7]](https://ngabbs.com/read.php?tid=17310135&_ff=-34587507&page=5#pid340241512Anchor) have shown the numeric value from $Pr(A_1)$ to $Pr(A_{99})$. However, the link in reference [[7]](https://ngabbs.com/read.php?tid=17310135&_ff=-34587507&page=5#pid340241512Anchor) does not provide a formula. Theoretically,

$$
\begin{equation}
Pr(A_i)=\left\{
\begin{aligned}
& (0.98)^{i-1} \times 0.02&1 \le i\le 50 & \\
& (0.98)^{50} \times (\prod_{k=51}^{i-1} \frac{99-k}{50}) \times \frac{i-49}{50} & 50<i\le 98 && \ where\ i \in N_+\\
& (0.98)^{50} \times \prod_{k=1}^{48} \frac{49-k}{50} & i = 99 & \\
\end{aligned}
\right.
\end{equation}
$$

With the results above, we can obtain the expectation of the number of pulls to get a 6★ operator, i.e., $\sum_{i=1}^{99}i \cdot Pr(A_i)$. 

This expectation is approximately **34.59**, or saying, the probability of getting a 6★ operator in Arknights is **2.89%**.

Here the value $34.59$ means that if we average all players' pull history (how many times did they pull to get a 6★ operator), the final result will be it. We can also understand this from a different aspect — if you pull for $N$ times, and you get 6★ operators for $m$ times. Each time you pull $n_i$ times to get a 6★ operator, where $i \in N_+,\ 1 \le i \le m$. The following equation will hold

$$
\begin{aligned}
\lim_{m \to +\infty} \frac {\displaystyle \sum_{i=1}^{m} n_i }{m}
=& \lim_{N \to +\infty} \frac {\displaystyle \sum_{i=1}^{m} n_i }{m}\\
=& \sum_{i=1}^{99}i \cdot Pr(A_i)\\
\approx& 34.59
\end{aligned}
$$

In fact, according to [[8]](https://ngabbs.com/read.php?tid=17310135&_ff=-34587507), the probability of getting a 6★ operator on the current pull is only decided by whether you got one in your previous pull, which satisfies the Markov property (or called "memorylessness"). So, we can also use Markov Chain to build the model for this question. Each state in the chain has the same meaning of $B_i$, and there are $100$ different states in this chain.

![](.\img\Arknights Markov Chain Model.png)

### Target 6★ Operator Drop: Attempts and Approaches of Calculating

It seems not so straightforward to calculate the theoretical probability of getting a **specific (target)** 6★ operator. Since if you want to get the probability of getting your target 6★ operator at your $N^{th}$ pull, it is possible that you got several other 6★ characters and reset the pity system many times until you get what you want at your last pull; it is also possible that you did not get any 6★ operators before your $N^{th}$ pull. As you can see, there exists a lot of mutually exclusive scenarios. And we need to calculate the probability of each scenario and sum them together.

To have a clearer sense of how many scenarios existing, let's count them based on how many times that you get a non-target 6★ operators from your first pull to your $(N-1)^{th}$ pull. We know in advance that the only one target 6★ operator is obtained in $N^{th}$ pull. So, on each individual pull before $N^{th}$ pull, you can whether get a non-target 6★ operator or not. It is possible that you did not get any non-target 6★ operator in $N-1$ pulls, corresponding to $\binom{N-1}{0}$ scenarios; it is also possible that you only get one non-target 6★ operator, which has $\binom{N-1}{1}$ different situations; similarly, getting 2 non-target 6★ operators corresponding to $\binom{N-1}{2}$ scenarios. We will find that the number of all exclusive scenarios is

$$
\sum_{i=0}^{N-1}\binom{N-1}{i} = 2^{N-1}
$$

which means, if we take this intuitive approach to get the theoretical probability, we need to calculate the probabilities of $2^{N-1}$ different scenarios. All of these indicate a terrible amount of work.

#### Promising Theoretical Approaches

Some promising methods of obtaining these accurate theoretical value of $Pr(S_i)$ are using Markov Chain and/or Dynamic Programming [[9]](https://bbs.nga.cn/read.php?tid=21471092&fav=6535953a). In [[9]](https://bbs.nga.cn/read.php?tid=21471092&fav=6535953a), the value of $Pr(S_i),\ i \in N_+,\ 1\le i \le 300$ were obtained. The author did not calculate the $Pr(S_i),\ i \in N_+,\ i \ge 300$ due to the Spark System. However, other pointed out that the index in the author's code was incorrect [[10]](https://bbs.nga.cn/read.php?pid=421121463&opt=128), and this method can not obtain the formulas of $Pr(S_i)$.

#### This Repo's Approach: Monte Carlo Simulation

I adopt the Monte Carlo simulation to estimate the theoretical $Pr(S_i)$ in this repo.

To be specific, we define that a **trial** is pulling for one or more times until you get your target 6★ operator. Simulate $N$ pulls ($N$ is big enough), and finally you get $n$ target 6★ operators (means that $n$ trials are finished). You pull $m_i$ times in each trial, namely $m_1,\ m_2,\ ...,\ m_n$. We regard each frequency as the corresponding estimated probability

$$
\frac{m_i}{n} \approx Pr(S_i),\ i \in N_+,\ 1 \le i \le n
$$

where there exists

$$
\begin{aligned}
&\lim_{n \to + \infty}\frac{m_i}{n}\\
=& \lim_{N \to + \infty}\frac{m_i}{n}\\
=&Pr(S_i),\ i \in N_+
\end{aligned}
$$

which indicates that in order to get a more accurate result, it is recommended to provided a larger value for command line arguments `-t` or `--total-pull-time` to expect a larger $n$.

> When the number of simulated pull reaches the limitation given by the command line argument `-t` or `--total-pull-time`, stop the simulation. It is possible that the last trial would be aborted because limitation is reached.

Now we take a look at how to calculate $Pr(W_i)$. According to the definitions of $W_i$ and $S_i$, we can see that $W_i$ means that **one** of $S_1$, $S_2$, ...​, $S_{i-1}$, $S_i$ happens. For example, $W_4$ means that you can get your target 6★ operator on your first pull ($S_1$), or second pull ($S_2$), or third pull ($S_3$) or fourth pull ($S_4$). When one of these scenarios happen, you just stop pulling, so we can avoid dealing with some complex events (e.g., $S_1$ happens then $S_3$ happens, or $S_3$ happens then $S_1$ happens, or $S_2$ happens twice, or $S_1$ happens for four times).

So we can conclude that

$$
\begin{aligned}
Pr(W_i)=& \sum_{k=1}^{i} Pr(S_k)\\
=& \sum_{k=1}^{i} \frac{m_k}{n},\ n\ is\ big\ enough
\end{aligned}
$$

# Generating the Random Numbers

Pulling in Arknights is a random experiment. To follow the same pulling rule in the simulation program, meanwhile ensuring the accuracy of the results, we need to generate a great number of high-quality uniformly distributed random numbers, and compare its value with several thresholds to decide whether we get a 6★/target 6★ operator. As a result, selecting a high-quality as well as blazing-fast random number generator (RNG) becomes the key part of the program.

### Pitfall: `rand() % n` is Evil!

It is quite common in many places that, we use the following code to generate a sequence of random number uniformly distributed in a given range, for example, $[0,\ 99]$

```c++
int seed;  // initialize seed with some value
srand(seed);
for(int i = 0; i < 16; ++i) {
    int rand_num = rand() % 100;
    cout << rand_num << endl;
}
```

> By the way, the code above, as well as some topics discussed below, is adopt from a lecture given by Stephan T. Lavavej from Microsoft^®^, which is mainly about generating random numbers in C++. If you are interested in this topic, I would strongly recommended you to take a look at this cool lecture. Here is the [link](https://channel9.msdn.com/Events/GoingNative/2013/rand-Considered-Harmful).
>

which is problematic!

Here is the problem. 

First, let's take a look at the period. In `glibc`, `rand()` uses Linear Congruential Generator (LCG) to produce random numbers. It may also use the Linear-Feedback Shift Register (LFSR) to generate the random number. According to [[11]](https://www.mscs.dal.ca/~selinger/random/), if LFSR is adopted, `rand()` only records last 32 output results. In other words, it can at most have $2^{32}$ different states; if LCG is adopted, the period is at most same as the modulus `m` that LCG algorithm used to generating numbers [[12]](https://en.wikipedia.org/wiki/Linear_congruential_generator). In fact, [[13]](http://sourceware.org/git/?p=glibc.git;a=blob;f=stdlib/random.c;hb=glibc-2.28#l287) and [[14]](http://sourceware.org/git/?p=glibc.git;a=blob;f=stdlib/random_r.c;hb=glibc-2.28#l353) show that, in `glibc` implementation, the value of `m` is $2^{31}-1$ (which is `0x7fffffff` in hexadecimal). What's more, the article from [[15]](https://webcache.googleusercontent.com/search?q=cache:47k2kqUbZO4J:https://www.eternallyconfuzzled.com/using-rand-c-c-advice-for-the-c-standard-librarys-rand-function/+&cd=4&hl=zh-CN&ct=clnk&gl=us), as well as [[16]](https://codeforces.com/blog/entry/61587), the period of `rand()` is shown to be roughly $2^{32}$. To estimate the $Pr(S_i)$, we at least need to simulate one billion (more than $2^{29}$) times of pull. This number would be increased to ten billion (more than $2^{33}$) to obtain a better accuracy. We can conclude that, the period of `rand()` is not enough for our large scale Monte Carlo simulation program.

> The original website of [[15]](https://webcache.googleusercontent.com/search?q=cache:47k2kqUbZO4J:https://www.eternallyconfuzzled.com/using-rand-c-c-advice-for-the-c-standard-librarys-rand-function/+&cd=4&hl=zh-CN&ct=clnk&gl=us) is down, so I provide the link cached by Google.

Second, the quality of the distribution of numbers generated by `rand()` and `rand() % n` is considered as low. No guarantees are made by the standard to ensure the quality of the random numbers generated by `rand()` [[17]](https://en.cppreference.com/w/c/numeric/random/rand). Actually, it is not truly uniformly distributed since it uses LCG or LFSR. **Even assuming that `rand()` is perfectly uniformly distributed, `rand() % n` is still not.** In this example from Stephan's lecture, assuming that numbers from `rand()` is perfectly uniform-distributed in $[0,\, 32767]$, and we want to get a uniform distribution number in $[0,\ 99]$, as code snip below

```c++
int src = rand();  // assume uniform [0, 32767]
int desired_range = 100;
int dist = src % desired_range;  // wrong!
```

This is a mistake is very easy to make. Why it is non-uniform distributed? The reason is actually very simple

```c++
int src = rand();  // assume uniform [0, 32767]
int desired_range = 100;
int dist = src % desired_range;
//          src     →     dist
//        [0, 99]   →   [0, 99]
//     [100, 199]   →   [0, 99]
//     [200, 299]   →   [0, 99]
// ...
// [32700, 32767]   →   [0, 67] -- problematic!
```

As you can see, the numbers in range $[0,\ 67]$ will have a slightly higher probability than the numbers in range $(67,\ 99]$. This error is introduced by the modulo. As long as the largest number that `rand()` can return (defined by `RAND_MAX`) is not a multiple of `desired_range`, this problem will be triggered. When estimating a $Pr(S_i)$ which itself has a very small value, this modulo-error will impose a great relative error to our results.

What's more, the following code is still not perfectly uniform distribution, because only when `src` is $99$ will make `dist` equal to $99$, however, for `dist` less than $99$ can has multiple corresponding `src` values.

```c++
int src = rand();  // assume uniform [0, 32767]
int desired_range = 100;
int dist = static_case<int>(src * 1.0 / RAND_MAX) * (desired_range - 1);
```
the following code is still wrong, since there is no way to uniformly map $32768$ integers to $100$ integers...
```c++
int src = rand();  // assume uniform [0, 32767]
int desired_range = 100;
int dist = static_case<int>(src * 1.0 / (RAND_MAX + 1)) * desired_range;
```

> In Stephan's lecture, he gives a more detailed explanation of these pitfalls. You can refer to his lecture if you are interested.

### Favored Approach - C++11 `<random>` Library

To achieve a high quality of uniform distribute number in order to reduce the inaccuracy of estimated probabilities as much as I can, I adopt the 64-bit Mersenne Twister Engine `mt19937_64` introduced in C++11 to generate the random numbers. The numbers produced by `mt19937_64` are filtered through `uniform_int_distribution`. Finally, we obtained the perfectly uniform-distributed numbers. **This approach eliminates, rather than just reduces, the errors discussed in previous section.** Compared the 32-bit version `mt_19937`, the 64-bit version Mersenne Twister Engine can accept more entropy, and its non-zero characteristic polynomial is roughly twice as many mas the 32-bit version [[18]](https://dl.acm.org/doi/pdf/10.1145/369534.369540).

A quick example code is shown below

```c++
#include <random>  // required header

std::mt19937_64 mt(42);  // 42 as the seed
std::uniform_int_distribution<unsigned int> dist(0, 99);  // an uniform distribution 
                                                          // on [0, 99] (both inclusive)
for (int i = 0; i < 100; ++i) {
    unsigned int ran = dist(mt);  // dist is driven by mt
    std::cout << rand_num << std::endl;
}
```

This example uses a fixed random seed, so it will have a deterministic results. You can create a `random_device` object and do the following

```c++
std::radom_device rd;
std::mt19937_64 mt(rd());
```

In this repo, to ensure the independency between each simulation, I use the `random_device` to generate a random seed, and then feed this seed to `mt19937_64`.

According to [[18]](https://dl.acm.org/doi/pdf/10.1145/369534.369540), Mersenne Twister Engine `mt19937_64` (as well as `mt19937`) has an extremely long period $2^{19937}-1$, you do not need to worry about the generated numbers will repeat again even if you run this simulation for a lifetime of universe. It also has a extremely high quality of distribution (ensured by the standard), which satisfy our requirements.

# Result Analysis
To save the space, I will only show the numeric value of $Pr(S_i), Pr(W_i)$, where $i \in N_+, 1 \le i \le 100$ here. I will also show several line charts based on a larger range of pulls. You can refer the full results where $i \in N_+, 1 \le i < 1000$ in the simulation output files under `/res` directory.

> If you do not know what do $Pr(S_i)$ and $Pr(W_i)$ mean, you can refer to the section [Abstracted Questions and Definitions](#abstracted-questions-and-definitions).

### Estimated Probability of Double-Rate-Up Limited Banner

The following results are obtained after $200$ billion simulated pulls, run with the command

```shell
./simulation_sequential --limited -t 200000000000 -n 2 -p 50
```

> You can also simply run with command
> ```shell
> ./simulation_sequential -t 200000000000
> ```
> which is the same.

> Again, it is recommended to redirect the output to a text file, since the output would be more than $1,000$ lines.

The following table shows the estimated numeric values of $Pr(S_i)$ and $Pr(W_i)$ of first $100$ pulls.

| $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ |
| ---- | --------- | --------- | ---- | --------- | --------- | ---- | --------- | --------- | ---- | --------- | --------- |
| 1    | 0.700%    | 0.700%    | 26   | 0.587%    | 16.693%   | 51   | 0.493%    | 30.110%   | 76   | 0.458%    | 52.817%   |
| 2    | 0.695%    | 1.395%    | 27   | 0.583%    | 17.276%   | 52   | 0.739%    | 30.849%   | 77   | 0.454%    | 53.270%   |
| 3    | 0.690%    | 2.085%    | 28   | 0.579%    | 17.855%   | 53   | 0.967%    | 31.816%   | 78   | 0.449%    | 53.719%   |
| 4    | 0.685%    | 2.771%    | 29   | 0.575%    | 18.430%   | 54   | 1.163%    | 32.979%   | 79   | 0.445%    | 54.164%   |
| 5    | 0.681%    | 3.451%    | 30   | 0.571%    | 19.001%   | 55   | 1.317%    | 34.296%   | 80   | 0.441%    | 54.605%   |
| 6    | 0.676%    | 4.127%    | 31   | 0.567%    | 19.568%   | 56   | 1.422%    | 35.717%   | 81   | 0.436%    | 55.041%   |
| 7    | 0.671%    | 4.798%    | 32   | 0.563%    | 20.131%   | 57   | 1.476%    | 37.194%   | 82   | 0.433%    | 55.474%   |
| 8    | 0.666%    | 5.464%    | 33   | 0.559%    | 20.689%   | 58   | 1.482%    | 38.675%   | 83   | 0.429%    | 55.902%   |
| 9    | 0.662%    | 6.126%    | 34   | 0.555%    | 21.245%   | 59   | 1.444%    | 40.120%   | 84   | 0.425%    | 56.327%   |
| 10   | 0.657%    | 6.783%    | 35   | 0.551%    | 21.796%   | 60   | 1.373%    | 41.493%   | 85   | 0.421%    | 56.748%   |
| 11   | 0.653%    | 7.436%    | 36   | 0.548%    | 22.343%   | 61   | 1.276%    | 42.769%   | 86   | 0.417%    | 57.165%   |
| 12   | 0.648%    | 8.084%    | 37   | 0.544%    | 22.887%   | 62   | 1.167%    | 43.936%   | 87   | 0.414%    | 57.579%   |
| 13   | 0.643%    | 8.727%    | 38   | 0.540%    | 23.427%   | 63   | 1.052%    | 44.988%   | 88   | 0.410%    | 57.989%   |
| 14   | 0.639%    | 9.366%    | 39   | 0.536%    | 23.963%   | 64   | 0.941%    | 45.929%   | 89   | 0.406%    | 58.395%   |
| 15   | 0.634%    | 10.000%   | 40   | 0.532%    | 24.495%   | 65   | 0.840%    | 46.770%   | 90   | 0.403%    | 58.798%   |
| 16   | 0.630%    | 10.630%   | 41   | 0.529%    | 25.023%   | 66   | 0.752%    | 47.521%   | 91   | 0.399%    | 59.197%   |
| 17   | 0.626%    | 11.256%   | 42   | 0.525%    | 25.548%   | 67   | 0.678%    | 48.200%   | 92   | 0.395%    | 59.593%   |
| 18   | 0.621%    | 11.877%   | 43   | 0.521%    | 26.069%   | 68   | 0.619%    | 48.819%   | 93   | 0.392%    | 59.984%   |
| 19   | 0.617%    | 12.494%   | 44   | 0.517%    | 26.587%   | 69   | 0.573%    | 49.393%   | 94   | 0.388%    | 60.373%   |
| 20   | 0.612%    | 13.107%   | 45   | 0.514%    | 27.101%   | 70   | 0.539%    | 49.931%   | 95   | 0.385%    | 60.757%   |
| 21   | 0.608%    | 13.715%   | 46   | 0.510%    | 27.611%   | 71   | 0.514%    | 50.445%   | 96   | 0.381%    | 61.139%   |
| 22   | 0.604%    | 14.319%   | 47   | 0.507%    | 28.118%   | 72   | 0.495%    | 50.940%   | 97   | 0.378%    | 61.517%   |
| 23   | 0.600%    | 14.919%   | 48   | 0.503%    | 28.621%   | 73   | 0.482%    | 51.422%   | 98   | 0.375%    | 61.892%   |
| 24   | 0.596%    | 15.514%   | 49   | 0.500%    | 29.121%   | 74   | 0.472%    | 51.894%   | 99   | 0.371%    | 62.263%   |
| 25   | 0.591%    | 16.106%   | 50   | 0.496%    | 29.617%   | 75   | 0.464%    | 52.358%   | 100  | 0.368%    | 62.631%   |

The following chart shows the estimated $Pr(S_i)$ and $Pr(W_i)$ of first $500$ pulls.

![line-chart-double-rate-up-limited-banner](img\line-chart-double-rate-up-limited-banner.png)

The blue line shows the estimated value of $Pr(S_i)$, which is assigned to the primary axis (on the left); the orange line shows the estimated value of $Pr(W_i)$, which is assigned to the secondary axis (on the right).

Here are some quick facts:

* on $58^{th}$ pull, you will have the biggest chance to get your target 6★ operator in a limited banner; $Pr(S_{58})=1.482\%$;
* if you prepared for $41$ pulls, you will have a $25\%$ chance of getting your target 6★ operator;
* if you prepared for $71$ pulls, you will have a $50\%$ chance of getting your target 6★ operator;
* if you prepared for $133$ pulls, this chance increases to $75\%$;
* you will have a $90\%$ chance to get your operator within $214$ pulls.

You can see that there are at least three observed peaks on the blue line. This is due to the pity system. The interesting thing is, although the odds of getting a 6★ operator starts to increase after $50$ pulls, the maximum value of $Pr(S_i)$ occurs when $i=58$, which is $8$ pulls after the pity system comes into effect. Here $Pr(S_{58})=1.482\%$. The second maximum $Pr(S_i)$ occurs when $i=116$, where $Pr(S_{116})=0.455\%$. The third maximum $Pr(S_i)$ occurs when $i=171$, where $Pr(S_{171})=0.199\%$.

Now we check the turning point (not the inflection point, same below) where $Pr(S_i)$ starts to increase rather than drops. Based on the numeric data and the chart above, we found that when $i = 52,\ 106,\ 168$, $Pr(S_i)$ is greater than its previous value, and keeps increasing for a while.

Note that in double-rate-up limited banners, the Spark System guarantees you to get the featured operator after $300$ pulls in the banner. However, the probability of this situation ($F_{300}$) is already very small — we can take a look at the estimated value of $Pr(S_{301})$, which requires the event $F_{300}$ to happen, according to the definition. Here the estimated $Pr(S_{301})=0.043 \%$, which equals to $Pr(F_{300}) \times Pr(Get\ the\ Target\ 6★\ Operator\ on\ 301^{st}\ Pull)$. Note that the second random event, $Get\ the\ Target\ 6★\ Operator\ on\ 301^{st}\ Pull$ does not care about the pulling results of previous pulls. We will have

$$
\begin{aligned}
\max Pr(F_{300}) & = \max \frac{Pr(S_{301})}{Pr(Get\ the\ Target\ 6★\ Operator\ on\ 301^{st}\ Pull)}\\
& = \frac{Pr(S_{301})}{\min Pr(Get\ the\ Target\ 6★\ Operator\ on\ 301^{st}\ Pull)}\\
& = \frac{0.043\%}{2\% \times 70\% \div 2}\\
& \approx 6.143\%
\end{aligned}
$$

, which means that the chance of not getting the target 6★ operator within $300$ pulls is less than $6.143\%$.

> There should exist a way to calculate $Pr(F_i)$ based on the estimated values of $Pr(S_i)$ and $Pr(W_i)$. It may be $1-Pr(W_i)$. Currently I am not fully sure that I am right😭.

### Estimated Probability of Double-Rate-Up Standard Banner

The following results are obtained after $200$ billion simulated pulls, run with command

```shell
./simulation_sequential --standard -t 200000000000 -n 2 -p 50
```

> You can also simply run with the command
> ```shell
> ./simulation_sequential --standard -t 200000000000
> ```
> which is the same.
>

The following table shows the estimated numeric value of $Pr(S_i)$ and $Pr(W_i)$ of first $100$ pulls.

| $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ |
| ---- | --------- | --------- | ---- | --------- | --------- | ---- | --------- | --------- | ---- | --------- | --------- |
| 1    | 0.500%    | 0.500%    | 26   | 0.441%    | 12.221%   | 51   | 0.389%    | 22.559%   | 76   | 0.406%    | 40.375%   |
| 2    | 0.497%    | 0.998%    | 27   | 0.439%    | 12.660%   | 52   | 0.565%    | 23.125%   | 77   | 0.403%    | 40.778%   |
| 3    | 0.495%    | 1.493%    | 28   | 0.437%    | 13.097%   | 53   | 0.730%    | 23.855%   | 78   | 0.400%    | 41.177%   |
| 4    | 0.493%    | 1.985%    | 29   | 0.435%    | 13.531%   | 54   | 0.872%    | 24.726%   | 79   | 0.397%    | 41.575%   |
| 5    | 0.490%    | 2.476%    | 30   | 0.432%    | 13.964%   | 55   | 0.984%    | 25.710%   | 80   | 0.395%    | 41.969%   |
| 6    | 0.488%    | 2.963%    | 31   | 0.430%    | 14.394%   | 56   | 1.062%    | 26.772%   | 81   | 0.392%    | 42.362%   |
| 7    | 0.485%    | 3.448%    | 32   | 0.428%    | 14.821%   | 57   | 1.104%    | 27.877%   | 82   | 0.390%    | 42.751%   |
| 8    | 0.483%    | 3.931%    | 33   | 0.426%    | 15.247%   | 58   | 1.111%    | 28.988%   | 83   | 0.387%    | 43.138%   |
| 9    | 0.481%    | 4.412%    | 34   | 0.424%    | 15.671%   | 59   | 1.088%    | 30.075%   | 84   | 0.385%    | 43.523%   |
| 10   | 0.478%    | 4.890%    | 35   | 0.422%    | 16.093%   | 60   | 1.040%    | 31.115%   | 85   | 0.382%    | 43.905%   |
| 11   | 0.476%    | 5.366%    | 36   | 0.419%    | 16.512%   | 61   | 0.974%    | 32.089%   | 86   | 0.380%    | 44.285%   |
| 12   | 0.473%    | 5.839%    | 37   | 0.417%    | 16.929%   | 62   | 0.898%    | 32.987%   | 87   | 0.378%    | 44.663%   |
| 13   | 0.471%    | 6.310%    | 38   | 0.416%    | 17.345%   | 63   | 0.819%    | 33.806%   | 88   | 0.375%    | 45.038%   |
| 14   | 0.469%    | 6.778%    | 39   | 0.413%    | 17.758%   | 64   | 0.741%    | 34.547%   | 89   | 0.372%    | 45.410%   |
| 15   | 0.466%    | 7.245%    | 40   | 0.411%    | 18.169%   | 65   | 0.671%    | 35.218%   | 90   | 0.370%    | 45.781%   |
| 16   | 0.464%    | 7.708%    | 41   | 0.409%    | 18.578%   | 66   | 0.610%    | 35.828%   | 91   | 0.368%    | 46.149%   |
| 17   | 0.461%    | 8.170%    | 42   | 0.407%    | 18.986%   | 67   | 0.558%    | 36.386%   | 92   | 0.366%    | 46.515%   |
| 18   | 0.459%    | 8.629%    | 43   | 0.405%    | 19.391%   | 68   | 0.517%    | 36.902%   | 93   | 0.363%    | 46.878%   |
| 19   | 0.457%    | 9.086%    | 44   | 0.403%    | 19.794%   | 69   | 0.485%    | 37.387%   | 94   | 0.361%    | 47.239%   |
| 20   | 0.454%    | 9.540%    | 45   | 0.401%    | 20.195%   | 70   | 0.461%    | 37.848%   | 95   | 0.359%    | 47.598%   |
| 21   | 0.452%    | 9.993%    | 46   | 0.399%    | 20.594%   | 71   | 0.443%    | 38.291%   | 96   | 0.357%    | 47.954%   |
| 22   | 0.450%    | 10.443%   | 47   | 0.397%    | 20.991%   | 72   | 0.431%    | 38.722%   | 97   | 0.354%    | 48.309%   |
| 23   | 0.448%    | 10.891%   | 48   | 0.395%    | 21.386%   | 73   | 0.422%    | 39.144%   | 98   | 0.352%    | 48.661%   |
| 24   | 0.445%    | 11.336%   | 49   | 0.393%    | 21.780%   | 74   | 0.415%    | 39.559%   | 99   | 0.350%    | 49.011%   |
| 25   | 0.443%    | 11.780%   | 50   | 0.391%    | 22.170%   | 75   | 0.410%    | 39.969%   | 100  | 0.348%    | 49.359%   |

The following chart shows the estimated $Pr(S_i)$ and $Pr(W_i)$ of first $500$ pulls.

<img src="img/line-chart-double-rate-up-standard-banner.png" alt="line-chart-double-rate-up-standard-banner"  />

The blue line shows the estimated value of $Pr(S_i)$, which is assigned to the primary axis (on the left); the orange line shows the estimated value of $Pr(W_i)$, which is assigned to the secondary axis (on the right).

Here are some quick facts:

* on $58^{th}$ pull, you will have the biggest chance to get your target 6★ operator in a limited banner; $Pr(S_{58})=1.111\%$;
* if you prepared for $55$ pulls, you will have a $25\%$ chance of getting your target 6★ operator;
* if you prepared for $102$ pulls, you will have a $50\%$ chance of getting your target 6★ operator;
* if you prepared for $189$ pulls, this chance increases to $75\%$;
* you will have a $90\%$ chance to get your operator within $306$ pulls.

You can see that in a double-rate-up standard banner, it is harder to get what you want than in a limited banner, since the conditional probability is lower ($25\%$ for each one of the featured 6★ operator in this kind of banner, compared with $35\%$ in a limited banner).

Similarly, due to the pity system, there also exist three turning points and three observed peaks in $Pr(S_i)$, which are

* turning points: $i=52,\ 105,\ 165$
* local maximum points and values:
  * $Pr(S_{58})=1.111\%,\ where\ i=58$
  * $Pr(S_{116})=0.431\%,\ where\ i=116$
  * $Pr(S_{171})=0.236\%,\ where\ i=171$

It is interesting that although the positions of the turning points are different from those in a double-rate-up limited banner, the positions of local maximum are the same.

### Estimated Probability of Single-Rate-Up Standard Banner

The following results are obtained after $200$ billion simulated pulls, run with command

```shell
./simulation_sequential --standard -t 200000000000 -n 1 -p 50
```

> You can also simply run with the command
> ```shell
> ./simulation_sequential --standard -t 200000000000 -n 1
> ```
> which is the same.

The following table shows the estimated numeric value of $Pr(S_i)$ and $Pr(W_i)$ of first $100$ pulls.

| $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ | $i$  | $Pr(S_i)$ | $Pr(W_i)$ |
| :--: | :-------: | :-------: | :--: | :-------: | :-------: | :--: | :-------: | :-------: | :--: | :-------: | :-------: |
|  1   |  1.000%   |  1.000%   |  26  |  0.778%   |  22.995%  |  51  |  0.605%   |  40.105%  |  76  |  0.453%   |  68.313%  |
|  2   |  0.990%   |  1.990%   |  27  |  0.770%   |  23.765%  |  52  |  0.956%   |  41.060%  |  77  |  0.446%   |  68.759%  |
|  3   |  0.980%   |  2.970%   |  28  |  0.762%   |  24.528%  |  53  |  1.278%   |  42.339%  |  78  |  0.439%   |  69.198%  |
|  4   |  0.970%   |  3.940%   |  29  |  0.755%   |  25.282%  |  54  |  1.553%   |  43.892%  |  79  |  0.433%   |  69.631%  |
|  5   |  0.961%   |  4.901%   |  30  |  0.747%   |  26.030%  |  55  |  1.766%   |  45.658%  |  80  |  0.428%   |  70.059%  |
|  6   |  0.951%   |  5.851%   |  31  |  0.740%   |  26.769%  |  56  |  1.908%   |  47.566%  |  81  |  0.422%   |  70.480%  |
|  7   |  0.942%   |  6.793%   |  32  |  0.732%   |  27.502%  |  57  |  1.977%   |  49.543%  |  82  |  0.416%   |  70.897%  |
|  8   |  0.932%   |  7.725%   |  33  |  0.725%   |  28.227%  |  58  |  1.975%   |  51.518%  |  83  |  0.411%   |  71.308%  |
|  9   |  0.923%   |  8.648%   |  34  |  0.718%   |  28.944%  |  59  |  1.913%   |  53.432%  |  84  |  0.406%   |  71.714%  |
|  10  |  0.913%   |  9.561%   |  35  |  0.710%   |  29.655%  |  60  |  1.801%   |  55.233%  |  85  |  0.401%   |  72.114%  |
|  11  |  0.904%   |  10.466%  |  36  |  0.703%   |  30.358%  |  61  |  1.656%   |  56.889%  |  86  |  0.395%   |  72.510%  |
|  12  |  0.895%   |  11.361%  |  37  |  0.696%   |  31.055%  |  62  |  1.492%   |  58.381%  |  87  |  0.390%   |  72.900%  |
|  13  |  0.886%   |  12.247%  |  38  |  0.689%   |  31.744%  |  63  |  1.322%   |  59.703%  |  88  |  0.385%   |  73.285%  |
|  14  |  0.877%   |  13.125%  |  39  |  0.683%   |  32.427%  |  64  |  1.159%   |  60.862%  |  89  |  0.380%   |  73.665%  |
|  15  |  0.869%   |  13.994%  |  40  |  0.676%   |  33.103%  |  65  |  1.010%   |  61.872%  |  90  |  0.375%   |  74.041%  |
|  16  |  0.860%   |  14.854%  |  41  |  0.669%   |  33.772%  |  66  |  0.880%   |  62.752%  |  91  |  0.370%   |  74.411%  |
|  17  |  0.851%   |  15.705%  |  42  |  0.662%   |  34.434%  |  67  |  0.773%   |  63.525%  |  92  |  0.366%   |  74.776%  |
|  18  |  0.843%   |  16.548%  |  43  |  0.656%   |  35.090%  |  68  |  0.686%   |  64.211%  |  93  |  0.361%   |  75.137%  |
|  19  |  0.834%   |  17.383%  |  44  |  0.649%   |  35.739%  |  69  |  0.619%   |  64.831%  |  94  |  0.356%   |  75.493%  |
|  20  |  0.826%   |  18.209%  |  45  |  0.643%   |  36.382%  |  70  |  0.570%   |  65.400%  |  95  |  0.352%   |  75.845%  |
|  21  |  0.818%   |  19.027%  |  46  |  0.636%   |  37.018%  |  71  |  0.533%   |  65.933%  |  96  |  0.347%   |  76.191%  |
|  22  |  0.810%   |  19.836%  |  47  |  0.630%   |  37.648%  |  72  |  0.506%   |  66.439%  |  97  |  0.342%   |  76.534%  |
|  23  |  0.801%   |  20.638%  |  48  |  0.624%   |  38.272%  |  73  |  0.487%   |  66.925%  |  98  |  0.338%   |  76.872%  |
|  24  |  0.794%   |  21.432%  |  49  |  0.617%   |  38.889%  |  74  |  0.473%   |  67.398%  |  99  |  0.334%   |  77.205%  |
|  25  |  0.786%   |  22.217%  |  50  |  0.611%   |  39.500%  |  75  |  0.461%   |  67.860%  | 100  |  0.329%   |  77.534%  |

The following chart shows the estimated $Pr(S_i)$ and $Pr(W_i)$ of first $500$ pulls.

![line-chart-single-rate-up-standard-banner](img/line-chart-single-rate-up-standard-banner.png)

The blue line shows the estimated value of $Pr(S_i)$, which is assigned to the primary axis (on the left); the orange line shows the estimated value of $Pr(W_i)$, which is assigned to the secondary axis (on the right).

Here are some quick facts:

* on $57^{th}$ pull, you will have the biggest chance to get your target 6★ operator in a limited banner; $Pr(S_{57})=1.977\%$;
* if you prepared for $29$ pulls, you will have more than $25\%$ chance of getting your target 6★ operator;
* if you prepared for $58$ pulls, you will have more than $50\%$ chance of getting your target 6★ operator;
* if you prepared for $93$ pulls, this chance increases to more than $75\%$;
* you will have more than $90\%$ chance to get your operator within $144$ pulls.

The single-rate-up standard banner is the easiest banner to get the featured 6★ operator — although the conditional probability is lower that that in a limited banner ($50\%$ compared with $70\%$), this kind of banner only has one featured operator, while a limited banner has two. Based on our assumption, the conditional rate shared by each operator featured by the limited banner is $70\% \div 2 = 35\%$, which is lower than $50\% \div 1 = 50\%$ in a single-rate-up banner.

However,in this kind of banner, there only exist two turning points and two observed peaks in $Pr(S_i)$ due to the pity system, which are

* turning points: $i=52,\ 106$
* local maximum points and values:
  * $Pr(S_{57})=1.977\%,\ where\ i=57$
  * $Pr(S_{116})=0.410\%,\ where\ i=116$

The reason why there are only two observed peaks might be that, in this type of banner, the odds of success is higher than other two types of banners ($50\%$ compared with $35\%$ and $25\%$), so the odds of failure is lower. Meanwhile, there exist two antagonism factors for the random event $S_i$. Let's call them $f(i)$ and $g(i)$. According to the definition, the occurrence of $S_i$ means the occurrence of $F_{i-1}$ **AND** the occurrence of random event $Get\ the\ Target\ 6★\ Operator\ on\ i^{th}\ Pull$ (again, this is different from $S_i$ — it does not care about the results of all the previous pulls). As the $i$ increases, the probability of the former will drop, since we need to ensure more pulls are failed, while the probability of the later will increase, since there will be "more chance" for the pity system coming into effect, or raising the probability to a higher level. So we have the following equations hold where $i$ belong to *some range*

$$
\begin{aligned}
f(i) &= Pr(F_i)\\
g(i) &= Pr(Get\ the\ Target\ 6★\ Operator\ on\ i^{th}\ Pull)\\
\Delta f(i) &= f(i+1) - f(i) < 0\\
\Delta g(i) &= g(i+1) - g(i) > 0\ ,\ where\ i\ belong\ to\ some\ range
\end{aligned}
$$

The results turn out that in this kind of banner, the speed of $f(i)$ dropping dominates the speed of $g(i)$ increasing when $i$ passes the second peak. So the value of $Pr(S_i)$ just keeps decreasing. As a result, the third peak disappears. However, you can find that second factor $g(i)$ did try to slow down the dropping of $Pr(S_i)$ around $161 \le i \le 176$.

This may also applicable to why there does not exist the fourth (or more) observed peaks of $Pr(S_i)$ in other two kinds of banners. Also may applicable to why the first maximum of $Pr(S_i)$ appears on $i=52$, rather than $i=51$, in all these three kinds of banners, although the pity system has already increased the odds of getting a 6★ operator on $i=51$.

> Note that in another simulation of $200$ billion pulls of this kind of banner, the estimated values of $Pr(S_{167})$ and $Pr(S_{168})$ are very close to each other (the difference is $0.000006\%$). It also might be the error of the simulation, although less likely. Without obtaining the theoretical formula of $Pr(S_i)$, it is hard to tell whether the third peak truly exists. So I used the word "observed" when analyzing these simulation results.

### Estimated Probability of Single-Rate-Up Limited Banner

This type of limited banner has not appeared in Arknights yet. However, you can refer to the simulation results in `./res/limited_200000000000_single_up_simu_1.res` if you are interested. Will not show the values of $Pr(S_i)$ and $Pr(W_i)$ here.

You can run this simulation with the following command if you would like do your own simulation

```shell
./simulaiton_sequential --limited -t <a-big-enough-number> -n 1 -p 50
```

> You can also run with
>
> ```shell
> ./simulation_sequential -t <a-big-enough-number> -n 1
> ```
>
> which is the same.

### Comparison Across Different Types of Banners

Let's draw the $Pr(S_i)$ and $Pr(W_i)$ of all these three kinds of banner in the same chart:

![line-chart-comparison-across-banners](img/line-chart-comparison-across-banners.png)

The solid lines are $Pr(S_i)$ and $Pr(W_i)$ from single-rate-up standard banner, the dotted lines are for the double-rate-up limited banner, and the dashed lines for the double-rate-up standard banner. As you can see, the $Pr(S_i)$ in single-rate-up standard banner increases and decreases fastest, while $Pr(S_i)$ in the double-rate-up standard banner is the slowest.

### Error Analysis

> There would be more metrics that can reasonably analyze the errors. I am not majored in statistics, and if you know more scientific way to measure the simulation error, I would be very appreciate if you can let me know.

Here, the theoretical value of $Pr(S_1)$ of these three kinds of banner can be easily obtained. Here I will compare the estimated values of $Pr(S_1)$ and its theoretical value to get the absolute error $\Delta Pr(S_1)$. And assume that the quality of the Mersenne Twister Engine `mt19937_64` is high enough, so that the absolute errors of all other $Pr(S_i)$ have the same absolute value as $Pr(S_1)$ has. Since the true value for $Pr(S_i)$ when $i > 50$ is hard to obtain, we will compare the estimated value with the absolute value of the absolute error $|\Delta Pr(S_1)|$ to have a sense of the accuracy of the estimated $Pr(S_i)$.

We say that if 

$$
\frac{|\Delta Pr(S_1)|}{Pr(S_i)} \le 5\%
$$

, we would think this estimated value is "accurate". $Pr(W_i)$ has the same "accurate" range since they are obtained based on the estimated $Pr(S_i)$.

> Note that since the true values of most of $Pr(S_i)$ are not known, the relative errors cannot be determined.

Theoretically, in double-rate-up limited banner,

$$
\begin{aligned}
Pr(S_1) &= 2\% \times 70\% \div 2\\
&= 0.7\%
\end{aligned}
$$

The estimated value of $Pr(S_1)$ is $0.699985\%$ (from `res/limited_200000000000_double_up_simu_2.res`). So the absolute error and its absolute value is

$$
\begin{aligned}
\Delta Pr(S_1) &= 0.699985\% - 0.7\% \\
               &= -0.000015\% \\
               &= -1.5 \times 10^{-7} \\
| \Delta Pr(S_1) | &= 1.5 \times 10^{-7}
\end{aligned}
$$

The threshold of an accurate $Pr(S_i)$ is

$$
\begin{aligned}
|\Delta Pr(S_1)| \div 5\% &= 3 \times 10^{-6} \\
                          &= 0.0003\%
\end{aligned}
$$

$Pr(S_i)$ needs to be equal or bigger than this threshold. After checking the data obtained from the simulation, all the estimated values of $Pr(S_i)$ and $Pr(W_i)$, where $1 \le i \le 731$, are accurate.

Similarly, according the simulation results of double-rate-up standard banner (from `res/standard_200000000000_double_up_simu_2.res`), the estimated values of  $Pr(S_i)$ and $Pr(W_i)$, where $1 \le i \le 649$, are accurate.

Finally, according the simulation results of single-rate-up standard banner (from `res/standard_200000000000_single_up_simu_3.res`), the estimated values of  $Pr(S_i)$ and $Pr(W_i)$, where $1 \le i \le 538$, are accurate.

Note that the method of analyzing error here may not be scientific enough to correctly show how truly accurate the results are. It is also possible that the estimated values that beyond this range are accurate enough, too.

You may find that the estimate $Pr(S_i)$ start to fluctuate when $i$ is relatively big (especially when $i$ is close to $1000$). Without the theoretical values (the true value) of $Pr(S_i)$, we cannot say that those estimated value, although less likely, is not accurate.

### Summary

Finally we summarize the simulation results of different kinds of banners for your quick reference.

|          Banner Type           | First Local Maximum $Pr(S_i)$ | Second Local Maximum $Pr(S_i)$ | Third Local Maximum $Pr(S_i)$ | $\ge 25\%$ Chance of Getting Your Operator | $\ge 50\%$ Chance of Getting Your Operator | $\ge 75\%$ Chance of Getting Your Operator | $\ge 90\%$ Chance of Getting Your Operator |
| :----------------------------: | :---------------------------: | :----------------------------: | :---------------------------: | :----------------------------------------: | :----------------------------------------: | :----------------------------------------: | :----------------------------------------: |
| Double-Rate-Up Limited Banner  |     $Pr(S_{58})=1.482\%$      |     $Pr(S_{116})=0.455\%$      |     $Pr(S_{171})=0.199\%$     |                 $41$ Pulls                 |                 $71$ Pulls                 |                $133$ Pulls                 |                $214$ Pulls                 |
| Double-Rate-Up Standard Banner |     $Pr(S_{58})=1.111\%$      |     $Pr(S_{116})=0.431\%$      |     $Pr(S_{171})=0.236\%$     |                 $55$ Pulls                 |                $102$ Pulls                 |                $189$ Pulls                 |                $306$ Pulls                 |
| Single-Rate-Up Standard Banner |     $Pr(S_{57})=1.977\%$      |     $Pr(S_{116})=0.410\%$      |         Not Observed          |                 $29$ Pulls                 |                 $58$ Pulls                 |                 $93$ Pulls                 |                $144$ Pulls                 |

Hope these results will help you in your next banner. Good Luck!

# Future Work

Due to the limited time, I did not carry out a much more larger simulation. Currently, the results presented in this article is based on $200,000,000,000$ simulated pulls, which needs about $35$ minutes to finish on single thread. I may execute a larger scaled parallel simulation and average the results to achieve a higher accuracy.

Besides, we can also try a different RNG. Arknights is made by Unity Engine, and we could try the RNG that the Unity adopts, or try a different seeding method (for example, seeding based on time and/or player's UID).

What's more, currently I did not try the same simulation configuration but with different `-p` value. We can also see with different pity system starting points, how the curve of $Pr(S_i)$ will change in its shape. This would also be an interesting question to have a look at — why the designer of the Arknights let the pity system starts after $50^{th}$ pull, rather than after $20^{th}$ pull or $30^{th}$ pull?

Finally, we can have a way to automatically generate the line charts and find the turning points as well as the maximum values. I may implemented this functionality using Python in the future.

# Acknowledgements

* The markdown code for the table of content is automatically generated by [ekalinin's tool](https://github.com/ekalinin/github-markdown-toc) — a very cool tool!

* Since I am not majored in statistics, the discussion in this article has a limited depth. If you have any ideas about the method that can obtain these theoretical value of $Pr(S_i)$ and $Pr(F_i)$, or simulation error analysis, or some mistakes in this article, please do not hesitate to let me know. I would be greatly appreciate it — actually I am very curios about the theoretical way to get the accurate results. Welcome to post your precious thoughts😀.

* There are several fantastic repos built by others, which inspired me a lot — thank you guys!

  💮[Frizu - Arknights 6-star Headhunting Rate Analysis](https://github.com/KaidenFrizu/GachaPull): in this repo the author did a large amount of research about the odds of getting a 6★ operator. Frizu has a series of articles [*Arknights 6★ Headhunting Rate Analysis*](https://rpubs.com/Frizu/arknightstheo), [*Arknights’ Non-6★ Headhunting Streak EDA*](https://rpubs.com/Frizu/arknightsgacha), that illustrates his method, the math and the results.

  💮[GalAster - ArkKnightsCapsule](https://github.com/GalAster/ArkKnightsCapsule): in this repo, the author estimated probabilities of getting one of the featured 6★ operators, or both of the featured 6★ operators in double-rate-up standard banner and double-rate-up limited banner. What's more, the odds of making the featured 6★ operators reach their max potential are also estimated.

  

  
<footer id="footer" class="main-content-wrap">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.2/MathJax.js?config=TeX-MML-AM_SVG"></script>
</footer>
