# PartiallyFD
This repository contains an R package for the Paper "Partially Observed Functional Data: The Case of Systematically Missing" by Dominik Liebl and Stefan Rameseder. Below, we present a demo for the installation of the package and the application of the estimator. 

The underlying statistical method is described in:  
[Partially Observed Functional Data: The Case of Systematically Missing](https://arxiv.org/abs/1711.07715)  
by Dominik Liebl and Stefan Rameseder (arXiv:1711.07715)

## Installation of required packages
```r
install.packages("devtools")
install.packages("fda")
install.packages("mclust")
library("mclust")
library("devtools")
```
### Installation of _PartiallyFD_-Package
```r
install_github("stefanrameseder/PartiallyFD")
library("PartiallyFD")
```
## Load data 
```r
data(log_partObsBidcurves)
attach(log_partObsBidcurves)

combinedNEG <- cbind(log_bc_fds[["NEG_HT"]], log_bc_fds[["NEG_NT"]])

matplot(x = md_dis, y=combinedNEG, ylim = c(0,10), # dim(combinedNEG)
        col = PartiallyFD:::addAlpha("black", 0.2), type = "l", lwd = 1,
        ylab = "", xaxt = "n", yaxt = "n", lty = "solid")
```
### Exclude outliers
```r
firstOutlier 	<- which.max(apply(combinedNEG, 2, max, na.rm = TRUE))
secondOutlier 	<- which.max(apply(combinedNEG[ ,-firstOutlier], 2, max, na.rm = TRUE))
combinedNEG_woOutlier <- combinedNEG[ ,-c(firstOutlier, secondOutlier+1)]

matplot(x = md_dis, y=combinedNEG_woOutlier, ylim = c(0,10), # dim(combinedNEG)
        col = PartiallyFD:::addAlpha("black", 0.2), type = "l", lwd = 1,
        ylab = "", xaxt = "n", yaxt = "n", lty = "solid")

```
## Exploratory Data Analysis (Figure 2)
```r
# Find components on "fully observed" domain [MW, 1830MW)
d_max       <- which.min(md_dis < 1830)
d_min       <- which.max(md_dis >= 0)

# Calculate Eigenvectors and Scores
X_mat       <- combinedNEG_woOutlier[d_min:d_max, ]
X_cent_mat  <- X_mat - rowMeans(X_mat)
Cov_mat     <- X_cent_mat %*% t(X_cent_mat) / ncol(X_cent_mat)
eigen_obj   <- eigen(Cov_mat)
c(cumsum(eigen_obj$values)/sum(eigen_obj$values))[1:5]

# Standardizing to L2-norm == 1 (assuming [a,b]=[0,1])
eigenvec_1  <- eigen_obj$vectors[,1] * sqrt(length(d_min:d_max))
eigenvec_1  <- eigenvec_1 * sign(sum(eigenvec_1))
PC_scores   <- c(eigenvec_1 %*% X_cent_mat)/length(d_min:d_max)

# Remove point-mass on minimal PC-scores (zero-functions)
quantile(PC_scores, seq(0, 0.1, 0.005))
thr         <- -5.16
scoreIndicesProbMass <- PC_scores <= thr
PC_scores_red <- PC_scores[PC_scores > thr]

# Cluster PC_scores_red
mclust.obj <- densityMclust(data = PC_scores_red, G=2) 
clust_vec  <- mclust.obj$classification

par(mfrow=c(1,1), mar=c(4.5,4,2.5,1)+0.1, family = "sans")
plotDensityMclust1(mclust.obj, xlab="First FPC-Scores (99.8% Explained Variance)", main="")
mtext(text = "Normal Mixture Cluster Result", side = 3, line = 1.25, cex=1.2)
hist(PC_scores_red, add=TRUE, freq = FALSE, breaks = 12)
points(x =              PC_scores_red[clust_vec==2], 
       y = rep(0,length(PC_scores_red[clust_vec==2])), col = alpha("red",1), bg=alpha("red",1),
       pch=21, cex=1)
points(x =              PC_scores_red[clust_vec==1], 
       y = rep(0,length(PC_scores_red[clust_vec==1])), col = alpha("blue",1), bg=alpha("blue",1), 
       pch=22, cex=1)
points(x = PC_scores[PC_scores < thr], 
       y = seq(0,0.1,len=length(PC_scores[PC_scores < thr])), col=alpha("darkorange",1), bg=alpha("darkorange",1),
       pch=23, cex=1)
legend("topleft", legend = c("High-Price Cluster", "Low-Price Cluster", "Zero-Functions"), 
       pt.cex=1,
       pch=c(21,22,23), 
       pt.bg = c(alpha("red",1), alpha("blue",1), alpha("darkorange",1)), 
       col=c(alpha("red",1), alpha("blue",1), alpha("darkorange",1)), bty="n")
dev.off()
```
### Compare the distribution with the Gaussian density
```r
## Histogram and Gaussian density:
g           <- PC_scores_red[clust_vec==2]
m           <- mean(g)
std         <- sqrt(var(g))
hist(g, prob=TRUE, breaks = 8, ylim=c(-0, 0.8), xlim = c(-.5,3), main="Histogram", xlab="")
curve(dnorm(x, mean=m, sd=std), lwd=2, add=TRUE, yaxt="n")
dev.off()
```

### Reduce Sample by excluding Low-Price and Zero-Function Clusters
```r
reducedDomSample <- combinedNEG_woOutlier[ , !scoreIndicesProbMass]
combinedNEG_woOutlier <- reducedDomSample[ , !scoreIndicesClust]

matplot(combinedNEG_woOutlier, type = "l", col = PartiallyFD:::addAlpha("black", 0.3))

combinedNEG_der <- cbind(log_bc_fds_der[["NEG_HT"]], log_bc_fds_der[["NEG_NT"]])
combinedNEG_woOutlier_der <- combinedNEG_der[ , -c(firstOutlier, secondOutlier+1)]
reducedDomSample <- combinedNEG_woOutlier_der[ , !scoreIndicesProbMass]
combinedNEG_woOutlier_der <- reducedDomSample[ , !scoreIndicesClust]

scoreIndicesClust <- clust_vec==1
paste0("Point mass at Zero: ",sum(scoreIndicesProbMass) ," and Low-Price Cluster:", sum(scoreIndicesClust))
```

## Application 
```r
maxBasisLength 		<- 51		# The Basis Selection Criterion in BIC 
basisSel		<- "Med" 	# The Basis Selection Criterion in BIC 
B			<- 1000 	# Number of Bootstrap Replications
basis_choice		<- "fourier"# The basis where the functions are projected onto
res 			<- calcFTC(fds = combinedNEG_woOutlier, comp_dom = md_dis,
                   		basis_seq = seq(3,maxBasisLength,2), base = basis_choice,
                   		maxBasisLength = maxBasisLength, basisChoice = basisSel,
                   		alpha = 0.05, B = B, derFds = combinedNEG_woOutlier_der)
```

### Test-Procedure (See Section 3)
```r
PartiallyFD:::checkFtcHypothesis(res$romWolf$ent)
```

### Calculate Confidence Intervalls
```r
alpha           <- 0.05
nPerP           <- apply(combinedNEG_woOutlier, 1, function(x) sum(!is.na(x))) 
ftcSD           <- sqrt(diag(res$ftcCov))/sqrt(nPerP)
critValue       <- qnorm(1-(alpha/2))
CISummand       <- critValue * ftcSD
ftcCI_plus      <- res$ftcMean + CISummand
ftcCI_minus     <- res$ftcMean - CISummand

krausSD         <- sqrt(diag(res$krausCov))/sqrt(nPerP)
CISummand       <- critValue * krausSD
krausCI_plus    <- res$krausMean + CISummand
krausCI_minus   <- res$krausMean - CISummand
```

### Final Plot with Estimates (Figure 3)
```r
p.cex 		<- 2
p_seq 		<- seq(1,p,8)
p_seq_main 	<- seq(1,p, 16)
m

layout(matrix(c(1,1,2), nrow = 1, ncol = 3, byrow = TRUE))
par(mar=c(5.1, 5.1, 2.1, 1.1))
matplot(x = 0, y = 0, 
	type = "l", 
	col = addAlpha("black", 0.2), 
	ylim = c(0, 10), xlim = c(0, 2500), 
	lty = 1, xaxt = "n", yaxt = "n", 
	ylab = "Log Price [Log(Euro/MW)/week]", 
	xlab = "Electricity Demand [MW]", 
	cex.lab=scl.axs+0.45)

abline(v= d_t[["NEG_HT"]], col = "lightblue")
matplot(x = md_dis, y = combinedNEG, type = "l", col = addAlpha("black", 0.2), ylim = c(0, 10), lty = 1, add = TRUE)

points(x = md_dis[p_seq_main], y = res$ftcMean[p_seq_main], col = "blue", pch = 16, cex = p.cex)
lines(x = md_dis, y = res$ftcMean, col = "blue", lwd = 2)
points(x = md_dis[p_seq_main], y = res$krausMean[p_seq_main], col = "darkred", pch = 17, cex = p.cex)
lines(x = md_dis, y = res$krausMean, col = "darkred", lwd = 2)

polygon(x = c(md_dis, rev(md_dis)),
        y = c(ftcCI_plus, rev(ftcCI_minus)),
        col = addAlpha("blue", 0.2), border = "blue", lwd = 1)
polygon(x = c(md_dis, rev(md_dis)),
        y = c(krausCI_plus, rev(krausCI_minus)),
        col = addAlpha("darkred", 0.2), border = "darkred", lwd = 1)

yLim <- c(5,10)
xLim <- c(1750, 2500)

rect(xLim[1],yLim[1],xLim[2],yLim[2],col = rgb(0.5,0.5,0.5,1/4))
axis(side = 1, at = seq(0, 2500, 500),  labels =  paste0(seq(0, 2500, 500), " MW"), cex.axis = scl.axs)	
axis(side = 2, at = seq(0, 10, 2),  labels =  paste0(seq(0, 10, 2)), cex.axis = scl.axs)	
legend(	"topleft", col = c("black", "darkred", "blue", "lightblue"), inset = 0.01, cex=scl.axs+ 0.4,
        lwd=c(1,2, 2, 3), lty = c("solid", "solid", "solid","solid"),
        pch=c(NA,17,16, NA), pt.cex=rep(p.cex*1.5,4),  
        legend = c(expression(paste(X[t])), expression(paste(hat(mu))), expression(paste(hat(mu)[FTC])), expression(paste(d[i]))))
matplot(x = 0, y = 0, type = "l", col = addAlpha("black", 0.2), ylim = c(0, 10), xlim = c(0, 2500), lty = 1, xaxt = "n", yaxt = "n",
        ylab = "Log Price in (Log Euro/MW)/week", xlab = "Electricity Demand in MW", cex.lab=scl.axs+0.45, add = TRUE)

par(mar=c(5.1, 3.1, 2.1, 2.1))		
matplot(x = md_dis, y = combinedNEG, type = "l", col = addAlpha("black", 0.2), lty = 1, 
        xaxt = "n", yaxt = "n", ylim = yLim, xlim = xLim,
        ylab = "", xlab = "", cex.lab=scl.axs+0.45)
abline( v= d_t[["NEG_HT"]], col = "lightblue", lwd = 1.5)

points(x = md_dis[p_seq], y = res$ftcMean[p_seq], col = "blue", pch = 16, cex = p.cex)
lines(x = md_dis, y = res$ftcMean, col = "blue", lwd = 2)
points(x = md_dis[p_seq], y = res$krausMean[p_seq], col = "darkred", pch = 17, cex = p.cex)
lines(x = md_dis, y = res$krausMean, col = "darkred", lwd = 2)

polygon(x = c(md_dis, rev(md_dis)),
        y = c(ftcCI_plus, rev(ftcCI_minus)),
        col = addAlpha("blue", 0.2), border = "blue", lwd = 1)
polygon(x = c(md_dis, rev(md_dis)),
        y = c(krausCI_plus, rev(krausCI_minus)),
        col = addAlpha("darkred", 0.2), border = "darkred", lwd = 1)
axis(side = 1, at = seq(xLim[1], xLim[2], 250),  labels =  paste0(seq(xLim[1], xLim[2], 250), " MW"), cex.axis = scl.axs)	
axis(side = 2, at = seq(yLim[1], yLim[2], 1),  labels =  paste0(seq(yLim[1], yLim[2], 1)), cex.axis = scl.axs)	
matplot(x = md_dis, y = combinedNEG, type = "l", col = addAlpha("black", 0.3), ylim = yLim, xlim = xLim, lty = 1, add = TRUE)


## Plot Covariance Estimates
# install.packages("rgl")
library(rgl)
persp3d(x = md_dis, y =md_dis, z = res$ftcCov, col="skyblue", zlim = c(-1,5))
persp3d(x = md_dis, y =md_dis, z = res$krausCov, col="skyblue", zlim = c(-1,5))
```
