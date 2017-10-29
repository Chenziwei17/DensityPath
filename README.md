# DensityPath
A novel algorithm, DensityPath, which accurately and efficiently reconstructs the underlying cell developmental trajectories for large-scale scRNAseq data.

Single cell RNAseq data from DensityPath analysis are often high dimensional large sample data. The scRNAseq data required by the Density path function is a single cell gene expression matrix. The rows of the matrix represent the cells, and the columns of the matrix represent the genes. We first need to reduce the dimensionality of the data, and densityPath has stability for the dimensionality reduction method. Diffusion map is used here to reduce dimension, and it needs to install "destiny" package. Secondly, the DensityPath algorithm is to find the density clusters to reconstruct the main path based on density clustering, so we need to install Topology Data Analysis(TDA) package, which contains the function of density clustering based on KDE density estimator. Finally, you need to install the package "gdistance", "raster" to calculate the geodesic distance on the surface, and the drawing packages "vegen" and "shape".

  >library("destiny")
  >library("TDA")
  >library("gdistance")
  >library("raster")
  >library("vegan")
  >library("shape")



1.1 Reduce the dimensionality of scRNAseq data.
Cell expression data are often high dimensional data. Using high-dimensional data directly will cause a lot of computing burden. Dimension reduction data not only reduces the dimensionality of the data to facilitate subsequent calculations but also reducing noise. There are many ways to reduce the dimension, such as: diffusion map, PCA, Isomap, etc. The density path method is robust to a variety of dimensionality reduction methods. In the following data analysis of this paper, we adopt the diffusion map method of non-linear dimensionality reduction and extract the first two-dimensional data for clustering.


  >if(ncol(XX)!=2){
    XX<-DiffusionMap(XX,k=2)
  }


1.2 Density clustering
The density of single-cell data is estimated using the k-nearest neighbor density estimator or Gaussian kernel density estimator. The density clustering is done by calling the clusterTree function in the TDA package, then get the density clusters idKDE. 


  >TreeKDE <- clusterTree(XX, k = k, h = h, density = "kde",
                         printProgress = FALSE)
  >densityKDE<-TreeKDE$density
  >idKDE<-setdiff(TreeKDE$id,TreeKDE$parent)
  >numKDEleaves<-length(idKDE)



1.3 Select high density clusters as the representative cell states. 
The peak point KDEdensitypeaks with the highest density value in each cluster is taken as the representative of the leaf clusters on density tree.

  >l<-1
  >KDEdensitypeaks<-matrix(1,numKDEleaves,2)
  >for (i in idKDE){
    clusterkde<-TreeKDE$DataPoints[[i]]
    KDEdensitypeaks[l,]<-XX[clusterkde[which.max(densityKDE[clusterkde])],]
    }


1.4 Construct the cell state-transition path. 

Divide the grid, calculate the density on the grid, forming a density surface. Using the costDistance function in the gdistance package to calculate the geodesic distance between any two density peaks of the mesh surface, the distance matrix dis is obtained.

  >xmin <-min(XX)
  >xmax<-max(XX)
  >ymin<-min(XX)
  >ymax<-max(XX)
  >numcell<-nrow(XX)
  >if (numcell<=10000)
  {
    numx<-101
    numy<-101
  }
  >if(numcell>10000)
  {
    numx<-501
    numy<-501
  }
  >Xlim <- c(xmin,xmax);  Ylim <- c(ymin, ymax);  
  >by <- (xmax-xmin)/(numx-1)
  >Xseq <- seq(Xlim[1], Xlim[2], by = by)
  >Yseq <- seq(Ylim[2],Ylim[1], by = -by)
  >Grid <- expand.grid(Xseq, Yseq)
  >KDE <- kde(X = XX, Grid = Grid, h = h)
  >r<-raster(nrows=numy, ncols=numx, xmn=xmin, xmx=xmax, ymn=ymin, ymx=ymax,crs="+proj=utm +units=m")
  >r[] <- KDE
  >T <- transition(r, function(x) mean(x), 8)
  >T <- geoCorrection(T)
  >C <-KDEdensitypeaks


Calculate the minimum spanning tree by calling the spantree function in the vegan package using the geodesic distance matrix.

  >spanningtree <- spantree(dis)


1.5 The following is the output of the graph.

  >par(mfrow = c(2,2))
  >par(mai = c(0.7,0.7,0.3,0.4))


#2-dimensional projection:

  >plot(XX, pch = 19, cex = 0.6, main = "2D mapping of single cell points",xlab="",ylab="")


#3-dimensional density surface:

  >Xseq <- seq(Xlim[1], Xlim[2], by = by)
  >Yseq <- seq(Ylim[1],Ylim[2], by = by)
  >Grid <- expand.grid(Xseq, Yseq)
  >zh<- matrix(KDE,ncol=length(Yseq),nrow=length(Xseq))

  >KDE <- kde(X = XX, Grid = Grid, h = h)
  >op <- par(bg = "white")
  >res<-persp(Xseq, Yseq, zh, theta = 345, phi = 30,
             expand = 0.5, col = "white",d =1,
             r=90,
             ltheta = 90,
             shade = 0, 
             ticktype = "detailed",
             #xlab = "X", ylab = "Y", zlab = "Sinc( r )" ,
             box = TRUE,
             border=NA,main="Density landscape",
             axes=F
  )

#Discrete density clusters:

  >plot(1:5,1:5,xlim=c(xmin,xmax),ylim=c(ymin,ymax),type="n",main="Density clusters", xlab="", ylab=" ") 
  >c<-1
  >for (i in idKDE){
    points(matrix(XX[TreeKDE[["DataPoints"]][[i]], ], ncol = 2), col = c,pch = 19, cex = 0.6)
    c<-c+1
  }


#density path:

  >p<-matrix(1,2,2)
  >plot(r,xlim=c(xmin,xmax),ylim=c(ymin,ymax),main="Density path",xlab="",ylab="")
  >KDEidspantree<-spanningtree$kid
  >numKDEedge<-numKDEleaves-1
  >KDEspantree<-matrix(1,numKDEedge,2)
  >KDEspantree[,1]<-as.matrix(2:numKDEleaves,numKDEleaves,1)
  >KDEspantree[,2]<-t(KDEidspantree)
  >for (i in 1:numKDEedge){
    p1<-KDEspantree[i,1]
    p2<-KDEspantree[i,2]
    p[1,]<-KDEdensitypeaks[p1,]
    p[2,]<-KDEdensitypeaks[p2,]
  #lines(p, col="red", lwd=2)
   >p1top2 <- shortestPath(T, p[1,], p[2,], output="SpatialLines")
    lines(p1top2, col="red", lwd=2)
  }
  >for (i in 1:numKDEleaves){
    points(KDEdensitypeaks[i,1],KDEdensitypeaks[i,2],pch = 19, cex = 1,col="blue")
  }

  >minspantreepath<-list()
  >for (i in 1:numKDEedge){
    p1<-KDEspantree[i,1]
    p2<-KDEspantree[i,2]
    p[1,]<-KDEdensitypeaks[p1,]
    p[2,]<-KDEdensitypeaks[p2,]
    #lines(p, col="red", lwd=2)
    p1top2 <- shortestPath(T, p[1,], p[2,], output="SpatialLines")
    adjpath<-as.matrix(((((p1top2@lines)[[1]])@Lines)[[1]])@coords)
    nr<-nrow(adjpath)
    a1<-min(adjpath[,1])-by
    a2<-max(adjpath[,1])+by
    b1<-min(adjpath[,2])-by
    b2<-max(adjpath[,2])+by
    Xseq1 <- seq(a1, a2, by = by)
    Yseq1 <- seq(b2, b1, by = -by)
    Grid1 <- expand.grid(Xseq1, Yseq1)
    KDE1 <- kde(X = XX, Grid = Grid1, h = h)
    pathpoints<-matrix(0,nrow(adjpath),3)
    pathpoints[,1:2]<-adjpath
  >for(m in 1:nr){
      l<-which(abs(adjpath[m,1]-Grid1[,1])+abs(adjpath[m,2]-Grid1[,2])==min(abs(adjpath[m,1]-Grid1[,1])+abs(adjpath[m,2]-    Grid1[,2])))
      pathpoints[m,3]<-KDE1[l]
    }
    minspantreepath<-c(minspantreepath,list(pathpoints))
  }


1.6 return value
Returns the density path, which contains the estimated density of the sample points, the two-dimensional coordinates of the density peaks, the geodesic distance between the density peaks, and the path of the minimum spanning tree on the three-dimensional density surface.

  >densitypath<-list()
  >densitypath<-c(densitypath,densityKDE=list(densityKDE))
  >densitypath<-c(densitypath,KDEdensitypeaks=list(KDEdensitypeaks))
  >densitypath<-c(densitypath,peaksdistance=list(dis))
  >densitypath<-c(densitypath,minspantreepath=list(minspantreepath))



