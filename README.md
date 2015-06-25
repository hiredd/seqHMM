seqHMM: Hidden Markov Models for Life Sequences and Other Multivariate, Multichannel Categorical Time Series
====================================================================================================

The seqHMM package is designed for the inference of hidden Markov models where both the hidden state space and the symbol space of observations are discrete and the observations consist of multiple sequences possibly with multiple channels (such as life history data with multiple life domains). Maximum likelihood estimation via EM algorithm and direct numerical maximization with analytical gradients is supported. All main algorithms are written in C++.

Package is still under development and should be available at CRAN in 2015.

If you have any questions or wishes, please contact Satu Helske or Jouni Helske, firstname.lastname (at) jyu.fi.

If you want to try the `seqHMM` package, you can install it via the `devtools` package:

```R
install.packages("devtools")
library(devtools)
install_github("helske/seqHMM")
```

Preview of the `seqHMM` package
---------------------------------------------------------------------------------

This example uses the `biofam` data from the `TraMineR` package. The data consist of a sample of 2000 individuals born between 1909 and 1972 constructed from the Swiss Household Panel (SHP) survey in 2002. The sequences consist of family life states from age 15 to 30 (in columns 10 to 25).


0 = "parents"
1 = "left"
2 = "married"
3 = "left+marr"
4 = "child"
5 = "left+child"
6 = "left+marr+child"
7 = "divorced"

For the functions of the `seqHMM` package, sequence data is given as a state sequence object (`stslist`) using the `seqdef` function in the `TraMineR` package. To show a more complex example, the original data is split into three separate channels. For the divorced state there is no information on children or residence, so these are assessed using the preceding states.

```
library(seqHMM)
library(TraMineR)

data(biofam)


# Complete cases in sex, birthyear, and first nationality
bio <- biofam[complete.cases(biofam[c(2:4)]),]

## Sequence data for the first six individuals
head(bio[10:25])

## Building one channel per type of event (married, children, or left)
bf <- as.matrix(bio[, 10:25]) 
married <- bf == 2 | bf== 3 | bf==6 
children <-  bf==4 | bf==5 | bf==6
left <- bf==1 | bf==3 | bf==5 | bf==6 | bf==7

## Giving labels, modifying sequences

# Marriage
married[married==TRUE] <- "Married" 
married[married==FALSE] <- "Single"
married[bf==7] <- "Divorced"

# Parenthood
children[children==TRUE] <- "Children" 
children[children==FALSE] <- "Childless"
# Divorced parents
div <- bf[(rowSums(bf==7)>0 & rowSums(bf==5)>0) | 
          (rowSums(bf==7)>0 & rowSums(bf==6)>0),]
children[rownames(bf) %in% rownames(div) & bf==7] <- "Children"

# Residence
left[left==TRUE] <- "Left home" 
left[left==FALSE] <- "With parents"
# Divorced living with parents (before divorce)
wp <- bf[(rowSums(bf==7)>0 & rowSums(bf==2)>0 & rowSums(bf==3)==0 &  rowSums(bf==5)==0 &  rowSums(bf==6)==0) | 
            (rowSums(bf==7)>0 & rowSums(bf==4)>0 & rowSums(bf==3)==0 &  rowSums(bf==5)==0 &  rowSums(bf==6)==0),]
left[rownames(bf) %in% rownames(wp) & bf==7] <- "With parents"

## Building sequence objects (starting at age 15)
marr.seq <- seqdef(married, start=15, alphabet=c("Single", "Married", 
                                                 "Divorced")) 
child.seq <- seqdef(children, start=15, alphabet=c("Childless", "Children")) 
left.seq <- seqdef(left, start=15, alphabet=c("With parents", "Left home"))

## Choosing colours for states
attr(marr.seq, "cpal") <- c("#E7298A", "#E6AB02", "#AB82FF")
attr(child.seq, "cpal") <- c("#66C2A5", "#FC8D62")
attr(left.seq, "cpal") <- c("#A6CEE3", "#E31A1C")
```
Multichannel sequence data are easily plotted using the `ssplot` function (ssplot for Stacked Sequence Plot).

```
## Plotting state distribution plots of observations
ssplot(list(marr.seq, child.seq, left.seq), type="d", plots="obs", 
       title="State distribution plots")
```                  
![ssp1](https://github.com/helske/seqHMM/blob/master/Examples/ssp1.png)

Multiple `ssp` objects can also be plotted together in a grid.

```
## Preparing plots for state distributios and index plots of observations for women
#  Sorting by scores from multidimensional scaling
ssp_f2 <- ssp(list(marr.seq[biofam$sex=="woman",],
                   child.seq[biofam$sex=="woman",],
                   left.seq[biofam$sex=="woman",]),
              type="d", plots="obs", border=NA,
              title="State distributions for women", title.n=FALSE,
              ylab=c("Married", "Children", "Left home"), 
              withlegend=FALSE, ylab.pos=c(1,2,1))
ssp_f3 <- ssp(list(marr.seq[biofam$sex=="woman",], 
                   child.seq[biofam$sex=="woman",],
                   left.seq[biofam$sex=="woman",]),
              type="I", sortv="mds.obs", plots="obs", 
              title="Sequences for women",
              ylab=c("Married", "Children", "Left home"), 
              withlegend=FALSE, ylab.pos=c(1.5,2.5,1.5))

## Preparing plots for state distributios and index plots of observations for men
ssp_m2 <- ssp(list(marr.seq[biofam$sex=="man",], 
                   child.seq[biofam$sex=="man",], 
                   left.seq[biofam$sex=="man",]),
              type="d", plots="obs", border=NA,
              title="State distributions for men", title.n=FALSE,
              ylab=c("Married", "Children", "Left home"), 
              withlegend=FALSE, ylab.pos=c(1,2,1))
ssp_m3 <- ssp(list(marr.seq[biofam$sex=="man",], 
                   child.seq[biofam$sex=="man",],
                   left.seq[biofam$sex=="man",]),
              type="I", sortv="mds.obs", plots="obs", 
              title="Sequences for men",
              ylab=c("Married", "Children", "Left home"), 
              withlegend=FALSE, ylab.pos=c(1.5,2.5,1.5))

## Plotting state distributions and index plots of observations for women and men 
## in two columns 
gridplot(list(ssp_f2, ssp_f3, ssp_m2, ssp_m3), cols=2, byrow=TRUE, 
           row.prop=c(0.42,0.42,0.16))
```
![gridplot](https://github.com/helske/seqHMM/blob/master/Examples/gridplot.png)

When fitting Hidden Markov models (HMMs), initial values for model parameters are first given to the `buildHMM` function. After that, the model is fitted with the `fitHMM` function using EM algorithm, direct numerical estimation, or a combination of both.

```
# Initial values for the emission matrices 
B_child <- matrix(NA, nrow=4, ncol=2) 
B_child[1,] <- seqstatf(child.seq[,1:4])[,2]+0.1 
B_child[2,] <- seqstatf(child.seq[,5:8])[,2]+0.1 
B_child[3,] <- seqstatf(child.seq[,9:12])[,2]+0.1 
B_child[4,] <- seqstatf(child.seq[,13:16])[,2]+0.1 
B_child <- B_child/rowSums(B_child)

B_marr <- matrix(NA, nrow=4, ncol=2) 
B_marr[1,] <- seqstatf(marr.seq[,1:4])[,2]+0.1 
B_marr[2,] <- seqstatf(marr.seq[,5:8])[,2]+0.1 
B_marr[3,] <- seqstatf(marr.seq[,9:12])[,2]+0.1 
B_marr[4,] <- seqstatf(marr.seq[,13:16])[,2]+0.1 
B_marr <- B_marr/rowSums(B_marr)

B_left <- matrix(NA, nrow=4, ncol=2) 
B_left[1,] <- seqstatf(left.seq[,1:4])[,2]+0.1 
B_left[2,] <- seqstatf(left.seq[,5:8])[,2]+0.1 
B_left[3,] <- seqstatf(left.seq[,9:12])[,2]+0.1 
B_left[4,] <- seqstatf(left.seq[,13:16])[,2]+0.1 
B_left <- B_left/rowSums(B_left)

# Initial values for the transition matrix 
A <- matrix(c(0.9,   0.06, 0.03, 0.01,
              0,    0.9, 0.07, 0.03, 
              0,      0,  0.9,  0.1, 
              0,      0,    0,    1), 
            nrow=4, ncol=4, byrow=TRUE)

# Initial values for the initial state probabilities 
initialProbs <- c(0.9, 0.07, 0.02, 0.01)

## Building the hidden Markov model with initial parameter values 
bHMM <- buildHMM(observations=list(child.seq, marr.seq, left.seq), 
                 transitionMatrix=A, 
                 emissionMatrix=list(B_child, B_marr, B_left),
                 initialProbs=initialProbs)

## Fitting the HMM 
HMM <- fitHMM(bHMM)
# or equivalently
HMM <- fitHMM(bHMM, em.control=list(maxit=100,reltol=1e-8), 
              itnmax=10000, method="BFGS")
```
A simple `plot` method is used to show an `HMModel` object as a graph. It shows hidden states as pie charts (vertices), with emission probabilities as slices and transition probabilities as arrows (edges). Initial probabilities are shown below the pies.

```
## Plot HMM
plot(HMM$model)
```
![HMMdefault](https://github.com/helske/seqHMM/blob/master/Examples/HMMdefault.png)

```

## A prettier version
plot(HMM$model, vertex.size=50, 
     # Thicker edges with varying curvature 
     cex.edge.width=3, edge.curved=c(0,-0.7,0.6,0,-0.7,0),
     # Show only states with emission probability greater than 0.1
     combine.slices=0.1, 
     # Label for combined states
     combined.slice.label="States with probability < 0.1",
     # Less space for legend
     legend.prop=0.3)
```
![HMM](https://github.com/helske/seqHMM/blob/master/Examples/HMModel.png)

The `ssplot` function can also be used for plotting the observed states and the most probable paths of hidden states of the HMM.

```
## Plotting observations and hidden states
ssplot(HMM$model, plots="both", sortv="mds.mpp", 
       xtlab=15:30, xlab="Age", title="Observed and hidden state sequences")
```
![sspboth_default](https://github.com/helske/seqHMM/blob/master/Examples/sspboth_default.png)
```
## Prettier version
ssplot(HMM$model, type="I",
                plots="both",
                # Sorting subjects according to multidimensional
                # scaling scores of the most probable hidden state paths
                sortv="mds.mpp", 
                # Naming the channels
                ylab=c("Children", "Married", "Left home"), 
                # Title for the plot
                title="Observed sequences and the 
most probable paths of hidden states",
                # Labels for hidden states (most common states)
                mpp.labels=c("1: Childless single, with parents", 
                             "2: Childless single, left home",
                             "3: Married without children",
                             "4: Married parent, left home"),
                # Colours for hidden states
                mpp.col=c("olivedrab", "bisque", "plum", "indianred"),
                # Labels for x axis
                xtlab=15:30,
                # Proportion for legends
                legend.prop=0.45)
```
![sspboth](https://github.com/helske/seqHMM/blob/master/Examples/sspboth.png)

The `logLik` and `BIC` functions are used for model comparison with the log-likelihood or the Bayesian information criterion (BIC).

```
## Likelihood
logLik(HMM$model)

# -4103.938

## BIC
BIC(HMM$model)

# 8591.137
```
The `trimHMM` function can be used to trim models by setting small probabilities to zero. Here the trimmed model led to model with slightly improved likelihood, so probabilities less than 0.01 could be set to zero.

```
## Trimming HMM
trimmedHMM <- trimHMM(HMM$model, maxit=100, zerotol=1e-02)
# "1 iterations used."
# "Trimming improved log-likelihood, ll_trim-ll_orig = 1.63542699738173e-06"

## Emission probabilities of the original HMM
HMM$model$emiss

# $`1`
# symbolNames
# stateNames  Childless      Children
# 1 1.00000000 1.454112e-177
# 2 1.00000000  3.134555e-21
# 3 1.00000000  1.597482e-14
# 4 0.01953035  9.804697e-01
# 
# $`2`
# symbolNames
# stateNames      Married     Single
# 1 1.220309e-16 1.00000000
# 2 1.602734e-02 0.98397266
# 3 9.833996e-01 0.01660043
# 4 9.464670e-01 0.05353301
# 
# $`3`
# symbolNames
# stateNames    Left home With parents
# 1 1.037512e-18 1.000000e+00
# 2 1.000000e+00 3.735228e-13
# 3 7.127756e-01 2.872244e-01
# 4 1.000000e+00 3.631100e-43

## Emission probabilities of the trimmed HMM
trimmedHMM$emiss

# $`1`
# symbolNames
# stateNames  Childless  Children
# 1 1.00000000 0.0000000
# 2 1.00000000 0.0000000
# 3 1.00000000 0.0000000
# 4 0.01953053 0.9804695
# 
# $`2`
# symbolNames
# stateNames    Married     Single
# 1 0.00000000 1.00000000
# 2 0.01603142 0.98396858
# 3 0.98340199 0.01659801
# 4 0.94646698 0.05353302
# 
# $`3`
# symbolNames
# stateNames Left home With parents
# 1 0.0000000    1.0000000
# 2 1.0000000    0.0000000
# 3 0.7127736    0.2872264
# 4 1.0000000    0.0000000
```
The `MCtoSC` function converts a multichannel model into a single channel representation. E.g. the `plot` function for `HMModel` objects uses this type of conversion. The `seqHMM` package also includes a similar function `MCtoSCdata` for merging multiple state sequence objects.

```
## Converting multichannel model to single channel model
scHMM <- MCtoSC(HMM$model)

ssplot(scHMM, sortv="from.end", sort.channel=0, legend.prop=0.45)
```
![scssp](https://github.com/helske/seqHMM/blob/master/Examples/scssp.png)




Coming later
---------------------------------------------------------------------------------------

<ul> 
 <li>Function for computing posterior probabilities</li>
 <li>Markov models</li>
 <li>Covariates (in the making)</li>
 <li>Simulating sequences from HMMs</li>
</ul> 



