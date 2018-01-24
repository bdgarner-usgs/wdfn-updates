---
author: Robert M. Hirsch
date: 2018-01-24
slug: Quantile-Kendall
draft: True
title: Daily Streamflow Trend Analysis
type: post
categories: Data Science
image: static/Quantile-Kendall/unnamed-chunk-8.1.png
 
 
author_gs: Jt5I-0gAAAAJ
 
author_staff: robert-hirsch
author_email: <rmhirsch@usgs.gov>

tags: 
  - R
  - EGRET
 
description: Using the R-package EGRET to investigate trends in daily streamflow
keywords:
  - R
  - EGRET
 
  - surface water
  - trends
---
Introduction
============

This document describes how to produce a set of graphics and perform the associated statistical tests that describe trends in daily streamflow at a single streamgage. The trends depicted cover the full range of quantiles of the streamflow distribution, from the lowest discharge of the year, through the median discharge, up through the highest discharge of the year. The method is built around the R package \[EGRET\] (Exploration and Graphics for RivEr Trends) (<https://CRAN.R-project.org/package=EGRET>). It makes it possible to consider trends over any portion of the time period for which there are daily streamflow records, and any period of analysis (the portion of the year to be considered, e.g. the entire year or just the summer months).

Getting Started
===============

First, you need to have installed and loaded the `EGRET` package. Then, you'll need need to create an `eList` object. See the `EGRET` vignette or user guide [here](http://pubs.usgs.gov/tm/04/a10/) for more information on how to load the data from any USGS streamgage or from a user supplied file of daily streamflow. See pages 4-7 of the user guide for more information.

For this post, we will use the Choptank River in Maryland as an example. There is an example data set included in `EGRET`. The data set consists of metadata, daily discharge data, and water quality data, but this application does not use the water quality data.

There are two limitation that users should know about this application before proceeding any farther. 1) The code was designed for discharge records that are complete (no gaps). 2) The discharge on every day should be a positive value (not zero or negative). The EGRET code that is used here to read in new data has a "work around" for situations where there are a very small number of non-positive discharge values. It adds a small constant to all the discharge data so they will all be positive. This should have almost no impact on the results provided the number of non-positive days is very small, say less than 0.1% of all the days. That translates to about 11 days out of 30 years. For data sets with more zero or negative flow days some different code would need to be written (we would appreciate it if an user could work on developing such a set of code).

To start, the following R commands are needed.

``` r
library(EGRET)
eList <- Choptank_eList
```

Just to get some sense about the data we will look a portion of the metadata (gage ID number, name, and drainage area in square kilometers) and also see a summary of the discharge data (discharge in in cubic meters per second).

``` r
print(eList$INFO$site.no)
```

    ## [1] "01491000"

``` r
print(eList$INFO$shortName)
```

    ## [1] "Choptank River"

``` r
print(eList$INFO$drainSqKm)
```

    ## [1] 292.6687

``` r
print(summary(eList$Daily$Date))
```

    ##         Min.      1st Qu.       Median         Mean      3rd Qu. 
    ## "1979-10-01" "1987-09-30" "1995-09-30" "1995-09-30" "2003-09-30" 
    ##         Max. 
    ## "2011-09-30"

``` r
print(summary(eList$Daily$Q))
```

    ##      Min.   1st Qu.    Median      Mean   3rd Qu.      Max. 
    ##   0.00991   0.93446   2.40693   4.08658   4.61565 246.35656

Loading the necessary packages and other R code
===============================================

To run the analysis and produce the graphs you will need a few R functions in addition to the `EGRET` package. You can copy the entire block of code shown below here and paste it into your workspace (all as a single copy and paste) or you can create an .R file from the code that you will source each time you want to use it. In addition to `EGRET` he functions use the packages \[`rkt`\] (<https://CRAN.R-project.org/package=rkt>) and [`zyp`](https://CRAN.R-project.org/package=zyp). You will have to make sure these are installed on your computer. You will also need to have the package [`Kendall`](https://CRAN.R-project.org/package=Kendall) installed because `zyp` depends on it.

``` r
library(rkt)
library(zyp)
################ this is the function you will use ##############
makeQuantileKendall <- function(eList, startDate = NA, endDate = NA, paStart = 4, paLong = 12, legendLocation = "topleft", legendSize = 1.5) {
  localDaily <- eList$Daily
  localINFO <- eList$INFO
  localINFO$paStart <- paStart
  localINFO$paLong <- paLong
  localINFO$window <- 30
  start <- if(is.na(startDate)) as.Date(localDaily$Date[1]) else as.Date(startDate)
  end <- if(is.na(endDate)) as.Date(localDaily$Date[length(localDaily$Date)]) else as.Date(endDate)
  localDaily <- subset(localDaily, Date >= start & Date <= end)
  eList <- as.egret(localINFO,localDaily)
  # eList <- setPA(eList,paStart = paStart, paLong = paLong, window = 30)
  # pdf(file = fileName, width = 8, height = 6)
  plotFlowSingleKendall(eList,istat = 1, qUnit = 2)
  eList <- setPA(eList,paStart=paStart,paLong=paLong,window=30) # NOW THE INDIVIDUAL STATS ARE ALL ON CLIMATE YEARS
  plotFlowSingleKendall(eList, istat = 4, qUnit = 2)
  plotFlowSingleKendall(eList, istat = 8, qUnit = 2)
  plotFlowSingleKendall(eList, istat = 5, qUnit = 2)
  # eList <- setPA(eList,paStart = paStart, paLong = paLong, window = 30)
  v <- makeSortQ(eList)
  sortQ <- v[[1]]
  time <- v[[2]]
  results <- trendSortQ(sortQ, time)
  pvals <- c(0.001,0.01,0.05,0.1,0.25,0.5,0.75,0.9,0.95,0.99,0.999)
  zvals <- qnorm(pvals)
  name <- eList$INFO$shortName
  ymax <- trunc(max(results$slopePct)*10)
  ymax <- max(ymax + 2, 5)
  ymin <- floor(min(results$slopePct)*10)
  ymin <- min(ymin - 2, -5)
  yrange <- c(ymin/10, ymax/10)
  yticks <- axisTicks(yrange, log = FALSE)
  p <- results$pValueAdj
  color <- ifelse(p <= 0.1,"black","snow3")
  color <- ifelse(p < 0.05, "red", color)
  pvals <- c(0.001,0.01,0.05,0.1,0.25,0.5,0.75,0.9,0.95,0.99,0.999)
  zvals <- qnorm(pvals)
  name <- paste0("\n", eList$INFO$shortName,"\n",
                start," through ", end, "\n", 
                setSeasonLabelByUser(paStartInput = paStart, paLongInput = paLong))
  plot(results$z,results$slopePct,col = color, pch = 20, cex = 1.0, 
       xlab = "Daily non-exceedance probability", 
       ylab = "Trend slope in percent per year", 
       xlim = c(-3.2, 3.2), ylim = yrange, 
       las = 1, tck = 0.02, cex.lab = 1.2, cex.axis = 1.2, 
       axes = FALSE, frame.plot=TRUE)
  mtext(name, side =3, line = 0.2, cex = 1.2)
  axis(1,at=zvals,labels=pvals, las = 1, tck = 0.02)
  axis(2,at=yticks,labels = TRUE, las = 1, tck = 0.02)
  axis(3,at=zvals,labels=FALSE, las = 1, tck=0.02)
  axis(4,at=yticks,labels = FALSE, tick = TRUE, tck = 0.02)
  abline(h=0,col="red")
  legend(legendLocation,c("> 0.1","0.05 - 0.1","< 0.05"),col = c("snow3",
              "black","red"),pch = 20,
         pt.cex=1.0, cex = legendSize)
}     
######################### what follows are a set of other functions neede #############
#
makeSortQ <- function(eList){
# creates a matrix called Qsort
# Qsort[dimDays,dimYears]
# no missing values, all values discharge values 
# sorted from smallest to largest over dimDays (if working with full year dimDays=365)
#   also creates other vectors that contain information about this array
#   
    localINFO <- getInfo(eList)
    localDaily <- getDaily(eList)
    paStart <- localINFO$paStart
    paLong <- localINFO$paLong
# determine the maximum number of days to put in the array
    numDays <- length(localDaily$DecYear)
    monthSeqFirst <- localDaily$MonthSeq[1]
    monthSeqLast <- localDaily$MonthSeq[numDays]
# creating a data frame (called startEndSeq) of the MonthSeq values that go into each year
    Starts <- seq(paStart, monthSeqLast, 12)
    Ends <- Starts + paLong - 1
    startEndSeq <- data.frame(Starts, Ends)
# trim this list of Starts and Ends to fit the period of record
    startEndSeq <- subset(startEndSeq, Ends >= monthSeqFirst & Starts <= monthSeqLast)
  numYearsRaw <- length(startEndSeq$Ends)
# set up some vectors to keep track of years
    good <- rep(0, numYearsRaw)
    numDays <- rep(0, numYearsRaw)
    midDecYear <- rep(0, numYearsRaw)
    Qraw <- matrix(nrow = 366, ncol = numYearsRaw)
  for(i in 1: numYearsRaw) {
    startSeq <- startEndSeq$Starts[i]
    endSeq <- startEndSeq$Ends[i]
    startJulian <- getFirstJulian(startSeq)
  # startJulian is the first julian day of the first month in the year being processed
  # endJulian is the first julian day of the month right after the last month in the year being processed
    endJulian <- getFirstJulian(endSeq + 1)
    fullDuration <- endJulian - startJulian
    yearDaily <- localDaily[localDaily$MonthSeq >= startSeq & (localDaily$MonthSeq <= endSeq), ]
    nDays <- length(yearDaily$Q)
    if(nDays == fullDuration) {
        good[i] <- 1
        numDays[i] <- nDays
        midDecYear[i] <- (yearDaily$DecYear[1] + yearDaily$DecYear[nDays]) / 2
        Qraw[1:nDays,i] <- yearDaily$Q
        }   else {
          numDays[i] <- NA
            midDecYear[i] <- NA
        }
    }
# now we compress the matrix down to equal number of values in each column
    j <- 0
    numGoodYears <- sum(good)
    dayCounts <- ifelse(good==1, numDays, NA)
    lowDays <- min(dayCounts, na.rm = TRUE)
    highDays <- max(dayCounts, na.rm = TRUE)
    dimYears <- numGoodYears
    dimDays <- lowDays
    sortQ <- matrix(nrow = dimDays, ncol = dimYears)
    time <- rep(0,dimYears)
    for (i in 1:numYearsRaw){
        if(good[i]==1) {
            j <- j + 1
            numD <- numDays[i]
            x <- sort(Qraw[1:numD, i])
            # separate odd numbers from even numbers of days
            if(numD == lowDays) {
              sortQ[1:dimDays,j] <- x
            } else {
            sortQ[1:dimDays,j] <- if(odd(numD)) leapOdd(x) else leapEven(x)
        }
          time[j] <- midDecYear[i]
        } 
    }
    
    sortQList = list(sortQ,time)
    
    return(sortQList)           
}
############################################
trendSortQ <- function(Qsort, time){
# note requires packages zyp and rkt
    nFreq <- dim(Qsort)[1]
    nYears <- length(time)
    results <- as.data.frame(matrix(ncol=9,nrow=nFreq))
    colnames(results) <- c("slopeLog","slopePct","pValue","pValueAdj","tau","rho1","rho2","freq","z")
    for(iRank in 1:nFreq){
        mkOut <- rkt(time,log(Qsort[iRank,]))
        results$slopeLog[iRank] <- mkOut$B
        results$slopePct[iRank] <- 100 * (exp(mkOut$B) - 1)
        results$pValue[iRank] <- mkOut$sl
        outZYP <- zyp.zhang(log(Qsort[iRank,]),time)
        results$pValueAdj[iRank] <- outZYP[6]
        results$tau[iRank] <- mkOut$tau
# I don't actually use this information in the current outputs, but the code is there 
# if one wanted to look at the serial correlation structure of the flow series      
        serial <- acf(log(Qsort[iRank,]), lag.max = 2, plot = FALSE)
        results$rho1[iRank] <- serial$acf[2]
        results$rho2[iRank] <- serial$acf[3]
        frequency <- iRank / (nFreq + 1)
        results$freq[iRank] <- frequency
        results$z[iRank] <- qnorm(frequency)    
    }
    return(results)
}
#############################################
#
getFirstJulian <- function(monthSeq){
    year <- 1850 + trunc((monthSeq - 1) / 12)
    month <- monthSeq - 12 * (trunc((monthSeq-1)/12))
    charMonth <- ifelse(month<10,paste("0",as.character(month),sep=""),as.character(month))
    theDate <- paste(year,"-",charMonth,"-01",sep="")
    Julian1 <- as.numeric(julian(as.Date(theDate),origin=as.Date("1850-01-01")))
    return(Julian1)
}
#
#######################################
# code for deleting one value when the period that contains Februaries
# has a length that is an odd number
leapOdd <- function(x){
    n <- length(x)
    m <- n - 1
    mid <- (n + 1) / 2
    mid1 <- mid + 1
    midMinus <- mid - 1
    y <- rep(NA, m)
    y[1:midMinus] <- x[1:midMinus]
    y[mid:m] <- x[mid1:n]
    return(y)}
#
#######################################
# code for deleting one value when the period that contains Februaries
# has a length that is an even number
leapEven <- function(x){
    n <- length(x)
    m <- n - 1
    mid <- n / 2
    y <- rep(NA, m)
    mid1 <- mid + 1
    mid2 <- mid + 2
    midMinus <- mid - 1
    y[1:midMinus] <- x[1:midMinus]
    y[mid] <- (x[mid] + x[mid1]) / 2
    y[mid1:m] <- x[mid2 : n]
    return(y)
}
#
########################################
odd <- function(x) {if ((x %% 2) == 0) FALSE else TRUE}
#
################# Calculation of what water year each day is in ########################
calcWY <- function (df) 
{
    df$WaterYear <- as.integer(df$DecYear)
    df$WaterYear[df$Month >= 10] <- df$WaterYear[df$Month >= 
        10] + 1
    return(df)
}
#
############ Calculating of what climate year each day is in ########################
calcCY <- function (df)
# computes climate year and adds it to the Daily data frame
{
  df$ClimateYear <- as.integer(df$DecYear)
  df$ClimateYear[df$Month >= 4] <- df$ClimateYear[df$Month >= 
                                                 4] + 1
  return(df)
}
#
#########################################################################
#
plotFlowSingleKendall <- function (eList, istat, yearStart = NA, yearEnd = NA, qMax = NA, 
                  printTitle = TRUE, tinyPlot = FALSE, customPar = FALSE, runoff = FALSE, 
                  qUnit = 2, printStaName = TRUE, printPA = TRUE, printIstat = TRUE, 
                  cex = 0.8, cex.axis = 1.1, cex.main = 1.1, lwd = 2, col = "black", ...){
  
  localAnnualSeries <- makeAnnualSeries(eList)
  localINFO <- getInfo(eList)
  qActual <- localAnnualSeries[2, istat, ]
  qSmooth <- localAnnualSeries[3, istat, ]
  years <- localAnnualSeries[1, istat, ]
  Q <- qActual
  time <- years
  LogQ <- log(Q)
  mktFrame <- data.frame(time,LogQ)
  mktFrame <- na.omit(mktFrame)
  mktOut <- rkt(mktFrame$time,mktFrame$LogQ)
  slope <- mktOut$B
  slopePct <- 100 * (exp(slope)) - 100
  slopePct <- format(slopePct,digits=2)
  pValue <- mktOut$sl
  pValue <- format(pValue,digits = 3)
  
  if (is.numeric(qUnit)) {
    qUnit <- qConst[shortCode = qUnit][[1]]
  } else if (is.character(qUnit)) {
    qUnit <- qConst[qUnit][[1]]
  }
  
  paLong <- localINFO$paLong
  paStart <- localINFO$paStart
  window <- localINFO$window
  
  qFactor <- qUnit@qUnitFactor
  yLab <- qUnit@qUnitTiny
  
  if (runoff) {
    qActual <- qActual * 86.4/localINFO$drainSqKm
    qSmooth <- qSmooth * 86.4/localINFO$drainSqKm
    yLab <- "Runoff in mm/day"
  } else {
    qActual * qFactor
    qSmooth <- qSmooth * qFactor
  }

  localSeries <- data.frame(years, qActual, qSmooth)
  
  if (!is.na(yearStart)){
    localSeries <- subset(localSeries, years >= yearStart)
  }
  
  if (!is.na(yearEnd)){
    localSeries <- subset(localSeries, years <= yearEnd)
  }
  
  yInfo <- generalAxis(x = qActual, maxVal = qMax, minVal = 0, 
                       tinyPlot = tinyPlot)
  xInfo <- generalAxis(x = localSeries$years, maxVal = yearEnd, 
                       minVal = yearStart, padPercent = 0, tinyPlot = tinyPlot)
  
  line1 <- localINFO$shortName
  nameIstat <- c("minimum day", "7-day minimum", "30-day minimum", 
                 "median daily", "mean daily", "30-day maximum", "7-day maximum", 
                 "maximum day")
  
  line2 <-  paste0("\n", setSeasonLabelByUser(paStartInput = paStart, 
                                      paLongInput = paLong), "  ", nameIstat[istat])

  line3 <- paste0("\nSlope estimate is ",slopePct,"% per year, Mann-Kendall p-value is ",pValue)
  
  if(tinyPlot){
    title <- paste(nameIstat[istat])
  } else {
    title <- paste(line1, line2, line3)
  }
  
  
  if (!printTitle){
    title <- ""
  }

  genericEGRETDotPlot(x = localSeries$years, y = localSeries$qActual, 
                      xlim = c(xInfo$bottom, xInfo$top), ylim = c(yInfo$bottom, 
                      yInfo$top), xlab = "", ylab = yLab, customPar = customPar, 
                      xTicks = xInfo$ticks, yTicks = yInfo$ticks, cex = cex, 
                      plotTitle = title, cex.axis = cex.axis, cex.main = cex.main, 
                      tinyPlot = tinyPlot, lwd = lwd, col = col, ...)
  lines(localSeries$years, localSeries$qSmooth, lwd = lwd, 
        col = col)
}
####################################
#  the following smoother function does the trend in real discharge units and not logs
#  it is placed here so that users wanting to run this alternative have it available
#
smoother <- function(xy, window){
  edgeAdjust <- TRUE
  x <- xy$x
  y <- xy$y
  n <- length(y)
  z <- rep(0,n)
  x1 <- x[1]
  xn <- x[n]
  for (i in 1:n) {
    xi <- x[i]
    distToEdge <- min((xi - x1), (xn - xi))
    close <- (distToEdge < window)
    thisWindow <- if (edgeAdjust & close) 
      (2 * window) - distToEdge
    else window
    w <- triCube(x - xi, thisWindow)
    mod <- lm(xy$y ~ x, weights = w)
    new <- data.frame(x = x[i])
    z[i] <- predict(mod, new)
  }
  return(z)
}
```

Running the makeQuantileKendall function
========================================

Now all we need to do is to run the **makeQuantileKendall** function that was the first part of the code we just read in. First we will run it in its simplest form, we will use the entire discharge record and our period of analysis will be the full **climatic year**. A climatic year is the year that starts on April 1 and ends on March 31. It is our default approach because it tends to avoid breaking a long-low flow period into two segments, one in each of two adjacent years. To run it in its simplest form the only argument we need is the eList (which contains the metadata and the discharge data).

``` r
makeQuantileKendall(eList)
```

<img src='/static/Quantile-Kendall/unnamed-chunk-4-1.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-4-2.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-4-3.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-4-4.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-4-5.png'/ title='TODO' alt='TODO' class=''/>

Explanation of the 5 basic plots.
=================================

The first plot is for the *minimum day*. The dots indicate the discharge on the minimum day of each climate year in the period of record. The solid curve is a smoothed representation of those data. It is specifically the smoother that is defined in the EGRET user guide (pages 16-18) with a 30-year window. For record as short as this one (only 32 years) it will typically look like a straight line or a very smooth curve. For longer records it can display some substantial changes in slope and even be non-monotonic. At the top of the graph we see two pieces of information. A trend slope expressed in percent per year and a p-value for the Mann-Kendall trend test of the data. The slope is computed using the Thiel-Sen slope estimator. It is discussed on pages 266-274 of Helsel and Hirsch, 2002, which can be found \[here\] (<https://pubs.usgs.gov/twri/twri4a3/>) although it is called the "Kendall-Theil Robust Line" in that text. It calculated on the logarithms of the discharge data and then transformed to express the trend in percent per year. The p-value for the Mann-Kendall test is computed using the adjustment for serial correlation introduced by in the zyp R package (David Bronaugh and Arelia Werner for the Pacific Climate Impacts Consortium (2013). zyp: Zhang + Yue-Pilon trends package. R package version 0.10-1. <https://CRAN.R-project.org/package=zyp>).

The second plot is for the *median* day of the year. The median day is computed for each year in the record. It is the middle day, that is 182 values had discharges lower than it and 182 values had discharges greater than it (for a leap year it is the average of the 183rd and 184th values ranked values). Everything else about it is the same as the first plot.

The third plot is for the *maximum day* of the year. Otherwise it is exactly the same in its construction as the first two plots. Note that this is the maximum daily average discharge and in general it will be smaller than the annual peak discharge, which is the maximum instantaneous discharge for the year, wereas this is the highest daily average discharge. For very large rivers the annual maximum day discharge tends to be very close to the annual peak discharge and can serve as a rough surrogate for it in a trend study. For a small stream, where discharges may rise or fall by a factor of 2 or more in the course of a day, these maximimum day values can be very different from the annual peak discharge.

The fourth plot is the *annual mean discharge*. It is constructed in exactly the same manner as the previous three, but it represents the mean streamflow for all of the days in the year rather than a single order statistic such as minimum, median, or maximum.

The fifth plot is something we call the *Quantile-Kendall plot*. Each plotted point on the plot is a trend slope (computed in the same manner as in the previous 4 plots) for a given order statistic. The point at the far left edge is the first order statistic, which is the annual minimum daily discharge. This is the result described on the first plot. The next point to its right is the trend slope for the second order statistic (second lowest daily discharge for each year), and it continues to the far right being the trend slope for the 365th order statistic (annual maximum daily discharge). Their placement with respect to the x-axis is based on the z-statistic (standard normal deviate) associated with that order statistic. It is called the daily non-exceedance probability. It is a scale used for convenience. It in no-way assumes that the data follow any particular distribution. It is simply used to provide the greatest resolution near the tails of the daily discharge distribution. The color represents the p value for the Mann-Kendall test for trend as described above. Red indicates a trend that is significant at alpha = 0.05. Black indicates an attained significance between 0.05 and 0.1. The grey dots are trends that are not significant at the alpha level of 0.1.

There is one special manipulation of the data that is needed to account for leap years (this is a detail about the computation but is not crucial to understanding the plots). The 366 daily values observed in a leap year are reduced by one so that all years have 365 values. The one value eliminated is accomplished by replacing the two middle values in the ranked list of values by a single value which is the average of the two. A similar approach is used when the period of analysis is any set of months that contains the month of February. The number of leap year values are reduced by one and the reduction takes place at the median value for the year.

Variations on the simplest example
==================================

The call to the function **makeQuantileKendall** with all of its arguments is this:

makeQuantileKendall(eList, startDate = NA, endDate = NA, paStart = 4, paLong = 12, legendLocation = "topleft", legendSize = 1.5)

Here is a list of the optional arguments that are available to the user.

**startDate** If we want to evaluate the trend for a shorter period than what is contained in the entire Daily data frame, then we can specify a different starting date. For example, if we wanted the analysis to start with Water Year 1981, then we would say: startDate = "1980-10-01". By leaving out the startDate argment we are requesting the analysis to start where the data starts (which in this case is 1979-10-01).

**endDate** If we want to evalute the trend for a period that ends before the end of the data set we can specify that with endDate. So if we wanted to end with Water Year 2009 we would say: endDate = "2009-09-30".

**paStart** is the starting month of the period of analysis. For example if we were interested in the trends in streamflow in a series of months starting with August, we would say paStart = 8. The default is paStart = 4, which starts in April.

**paLong** is the duration of the period of analysis. So, an analysis that covers all 12 months would have paLong = 12 (the default). A period that runs for the months of August - November would have paStart = 8 and paLong = 4. See pages 14 and 15 of the user guide for more detail on paStart and paLong.

**legendLocation** this argument simply determines where in the graph the legend is placed. It sometimes happens that the legend obscures the data so we may need to move it to another part of the graph. The can take on names such as "bottomleft", "topright" or "bottomright". The default is "topleft"

**legendSize** this argument determines the size of the legend. A value of 0 would cause the legend to vanish competely. The default is legendSize = 1.5. Typically one wouldn't want to go below about 0.75, unless one wanted no legend at all.

Here is an example bringing all of the arguments into play. It is for a case where we want to consider the time period 1982-08-01 through 2008-11-30, for only the months of August through November, and we want the legend to go in the bottom right and be only two-thirds the size of what we saw in the first example.

``` r
makeQuantileKendall(eList, 
                    startDate = "1982-08-01", endDate = "2008-11-30", 
                    paStart = 8, paLong = 4, 
                    legendLocation = "bottomright", legendSize = 1.0)
```

<img src='/static/Quantile-Kendall/unnamed-chunk-5-1.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-5-2.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-5-3.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-5-4.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-5-5.png'/ title='TODO' alt='TODO' class=''/>

Downloading the data for your site of interest
==============================================

The steps for downloading data from USGS web services (or obtaining it from user supplied information) and also for creating and saving the eList are described on pages 4-13 of the EGRET user guide. Here is a simple example:

Say we want to look at USGS station number 01646500 and we want to consider data from Climate Years 1921 through 2016. The following commands would do what is needed.

``` r
library(EGRET)
sta <- "05436500"
param <- "00060" # this is the parameter code for daily discharge
startDate <- "1920-04-01"
endDate <- "2016-03-31"
INFO <- readNWISInfo(siteNumber = sta, parameterCd = param, interactive = FALSE)
Daily <- readNWISDaily(siteNumber = sta, parameterCd = param, startDate = startDate, 
            endDate = endDate, verbose =  FALSE)
eList <- as.egret(INFO, Daily)
makeQuantileKendall(eList, legendLocation = "bottomleft")
```

<img src='/static/Quantile-Kendall/unnamed-chunk-6-1.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-6-2.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-6-3.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-6-4.png'/ title='TODO' alt='TODO' class=''/><img src='/static/Quantile-Kendall/unnamed-chunk-6-5.png'/ title='TODO' alt='TODO' class=''/>

Final thoughts
==============

This last set of plots is particularly interesting. What we see is that for the lowest 90 percent of the distribution, discharges have been rising over most of this record, and particularly so since about 1950. But in the highest 1% of the distribution discharges have been falling thoughout the record. As a consequence the overall variablity of streamflow in this agricultural watershed has generally been declining over time. This is generally thought to be related to the increasing use of conservation practices in the watershed.

It is worth noting that we can express the percentage changes per year in other ways than as percent per year, for example percentage change per decade or percentage change over the entire period of record. Take, for example, the estimated trend slope for the median in this last example. It is 0.68% per year. If we were to express that as percentage change per decade it would be 7% per decade (the change can be computed as 1.0068^10, or 1.070, which is a 7% increase). If we expressed it for the entire 96 year record it would be about a 92% increase over that period (computed as 1.0068^96, which is 1.9167 or and increase of about 92%).

These graphical tools are one way of summarizing what we might call a **signature of change** for any given watershed. It shows the changes, or lack of changes, that have taken place through all parts of the probability distribution of streamflow. This has potential as a tool for evaluating hydrologic or linked climate and hydrologic models. Observing the patterns on these graphs for the actual data versus the patterns seen when simulated data are used, can provide insights on the ability of the models used to project hydrologic changes into the future.

If you have questions or comments on the concepts or the implementation please contact the author <rhirsch@usgs.gov>.