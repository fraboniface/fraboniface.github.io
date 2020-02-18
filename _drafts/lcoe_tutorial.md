---
layout: post
title: Levelized Cost of Energy is a metric for investors, not for society
author: François
---


Investment firm Lazard has published the 13th version of their [Levelized Cost of Energy (LCOE) assessment](https://www.lazard.com/media/451086/lazards-levelized-cost-of-energy-version-130-vf.pdf).

It has then been cited out of context in an antinuclear report, and now I keep arguing over these numbers on Reddit with people who think we should ditch nuclear energy altogether because renewables appear so much cheaper in Lazard's estimates. Yet, my country (France) has cheap, clean, and reliable electriity thanks to paid-for nuclear power plants. I've also seen [this video](https://www.youtube.com/watch?v=cbeJIwF1pVY) (the whole channel is gold), that gives a good intuition of why nuclear power actually makes a lot of economic sense in the long run. The problem is not that it's too expensive, it's that it's unattractive to investors. Here I will show that LCOE is purely a metric for investors and should not be used to make societal choices.


## How Lazard computes those estimates and why they're suspicious
Even though LCOE has evident flaws, such as not accounting for variability beyond the capacity factor, nor the end of life of the plant, I was still surprised by the large discrepancy between the cheapest renewables and nuclear. So I had a thorough look at the document. I found a few assumptions that seemed off. To see how things changed if I modified some parameters, I reproduced their model in Python, because I'm much better at it than at Excel. [You can find the code on my GitHub](https://github.com/fraboniface/lazard_lcoe). I knew nothing about finance, so I learned just what I needed to know and asked a few questions to my friend and M&A analyst Paul. Now I got a model that works and things are pretty clear in my head.

### The methodology

Lazard's methodology is described in this slide:
![Lazard's method](/assets/img/lcoe/lazard_methodology.png)
Lazard builds a model of cash flows during the lifetime of the plant. On the right are assumptions that they have to make in order to estimate anything. The assumtions in blue are technology-dependent, while those in yellow are fixed for all technologies. At least that's what it looks like, because I actually had to change the MACRS depreciation schedule to 20 for the conventional technologies (non-renewables), in order to match their results.

The most important numbers are those of the capital structure. The company that operates the plant finances it via borrowing 60% of the required capital at 8% interest from a bank, and the remaining 40% comes from investors who want back 12% each year on average. It's a general rule: equity is always more expensive than debt (but you don't have to give back the initial capital your investors gave you). This debt structure defines the weighted average cost of capital, or WACC. It is defined as:
$$
WACC = P_{equity}C_{equity} + P_{debt}C_{debt}(1-t)
$$
, where $$P$$ indicates the percentage, $$C$$ the cost, and $$t$$ the tax rate, because debt is tax-deductible.

Let's now explain the model line by line. Just to be clear, monetary amounts are in millions, and negative amounts are in parenthesis.
- the first 3 lines including Year are part of the assumptions;
- Total Generation = Capacity x Capacity factor x 24 x 365 in MWh, /1000 in TWh;
- LCOE: what we're looking for, we have to assume a value for now;
- Total Revenues (millions): Total Generation in MWh x LCOE / 1,000,000;
- Total Fuel Cost: 0 for renewables, for conventional you have to multiply the heat rate with the fuel cost and divide by a billion to get something in M$/MWh, and then multiply the result by Total Generation in MWh;
- Total O&M: Fixed O&M x Capacity + Variable O&M x Total Generation, divided by a million. That's for year 1, and then you increase that by 2.25% every year;
- Total Operating Cost and EBITDA: the formulas they give are enough;
- Debt:
	1. get year 1 of Debt Oustanding by multiplying the capex by the percentage of debt (60%). Note that the capital cost already includes the construction time and accumulating debt over that period, as written in their doc;
	2. compute the next two lines using functions called IPMT and PPMT, assuming the loan duration is the lifetime of the facility. As we will see, there's something a bit fishy in these lines;
	3. Levelized Debt Service is the sum of the principal and interest payments;
	4. For the following years of Debt Outstanding, withdraw from year 1 the sum of the principal payments so far;
- Depreciation: it's an accounting trick that allows a company to count the loss of value of its material capital (here the production means) as cash loss, which paves the way to tax deductions. As we will see, it is very important;
- Taxable Income has a formula;
- Tax Benefit: how much tax you pay. Note that it's a liability, so you actually have to substract it in the calculation of the next line;
- After-Tax Net Equity Cash Flow: how much the investors earn each year. You can see in year 0 a number that corresponds to their initial investment, that is 40% of the capex.

Once you got all that filled up, you can compute the IRR of the investors. I'll come back to what it is, but in practice it amount to calling the IRR function on the last line. The goal is then to adjust the value of the LCOE to find a IRR equal to the assumed equity cost of 12%.


### Things I find suspicious
Even though I knew nothing about finance before this work, there are a number of things that make me believe that these estimates might not be as serious as they seem. I'm not saying we shouldn't trust them, but I don't think they're rock solid or exampt from bias.
- Why separate between renewable and conventional instead of low-carbon and fossil fuels? It sounds like they want to discard nuclear as a green option from the start. It may just be that the cost structure is more similar to fossil fuels plants, though;
- If you want to reproduce the exact same numbers as in the table above, you have to assume that the loan duration is the lifetime plus one year, and not the lifetime. If you do so, the loan is not completely paid for at the end of year 20. The numbers they show at year 20 are actually those of year 21. If you look at [their version 12 estimates](https://www.lazard.com/media/450784/lazards-levelized-cost-of-energy-version-120-vfinal.pdf), you see that the last principal payment is not equal to the outstanding debt. When you make the calculations, it's actually still the case in version 13, but they replaced year 20 with year 21;
- if you do what i just said to keep the exact same numbers, and compute the IRR of the last line, you find something like 8%, not 12% (and 54$/MWh is the LCOE they report for that technology). I used the numpy-financial IRR function. I also tried with that of Open Office Calc, same thing. I don't have Excel, though. I think that's why my estimates are consistently a few dollars above theirs;
- Why not make it clearer that the MACRS depreciation schedule is in the end technology-dependent? I don't think they manipulated anything because, in my understanding, it's set by law, but it clearly gives an advantage to renewables;
- their capacity factors for wind and solar are quite optimistic. They go from 21% to 34% for PV utily scale, and from 38% to 55% for onshore wind. Yet, looking at data from [ourworldindata](https://ourworldindata.org/renewable-energy), [the US Energy Information Administration](https://www.eia.gov/electricity/monthly/epm_table_grapher.php?t=epmt_6_07_b), or [the National Renewable Energy Laboratory](https://atb.nrel.gov/electricity/2018/index.html?t=su), 15%-25% for solar and 25%-50% for onshore wind would have been much more honest;
- from v12 to v13, they changed a few assumptions in a rather significant way  without further justification (in their defense, nothing is jutified). I specifically have in mind the 5-fold increase in variable O&M for nuclear. Since it is the plant that produces the most, it will have the largest impact. Not that I think the modification is necessarily wrong, but if after 12 iterations of that exercize, they still need to change numbers that much from a year to another (for a technology that's not moving as fast as renewables I mean), it's to me an indication that the initial work of finding the values to start from might not be rock solid;
- finally, the IRR is computed on the first 20 year no matter the lifetime of the project. It can seem very wrong, but we will see that it doesn't change much. Still, my M&A friend told me that's not correct.

Despite these pointing to the possibility of the producer of this report having a slight bias towards renewables, I think we can rely on these estimates. I just wanted you to have those in mind. My real point, that I will argue now, is that LCOE is not a good metric for long-term projects, whereas thinking long term is exactly what we need these days.