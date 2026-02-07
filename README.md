# Images To 3D

## Intro

The goal of this project is to develop a program that create a 3d model from some images. You take some photos of an 
object, and then, the program should generate a 3d mesh for the object  

## Summary

1. Use case description
   1. Requirements
   2. file format
2. Technical description
   1. Image preprocessing
   2. Find the seed
   3. Likelihood


## 1. Use case description

This part will describe how to use this program.

### 1.1. Requirements

If you want this program to create the 3d model correctly, you have to be careful at some details.

#### 1.1.1 The light

The light is something really useful to understand the 3D in an image. However, analyse it is pretty complicated. In 
fact, we have to know where does the light source come from, all the characteristics of this light source, and even
how does the object reflect this light. And all this information are really hard to get.  
So, we decide to ignore the light. We prefer have less light disturbance possible. In order that this program execute 
at his best, you should try to put a uniform light on the objects.

#### 1.1.2 The germ

The algorithm used to create the voxel use a germ. In other words, we take a first cube in the 3D object, and then we 
add cube per cube until we have something really similar to the object.  
The problem is on the choice of the first element. We decided to chose it via a reference. The reference is an image 
that you have to put on the object. We shall be able to see this image in all the photos taken.  
This image has also another utility. It permits to know what is the relative orientation of the camera. 

### 1.2. The file format

The program will output a .vox file. It is a file format that stores voxels. You can then view the result, or transform
in another file format with https://imagetostl.com/view-vox-online.

## 2. Technical description


### 2.1. Image preprocessing

#### 2.1.1 Removing light

We want to remove the light, and to have in image as if there was a complete uniform light.  
We will suppose that it is lighted with a white light, with no reflections. We will approximate the effect of light on 
a surface as a scalar multiplication. So, to erase that scalar, we will normalize the color, and then multiply it by 255.  
**The problems:**
- Two different colors can be considered as same.
- If the light source is not white, it can be completely unuseful


### 2.2. Find the seed

This part will search the reference image in the image, and then will deduct the camera position from the reference.

#### 2.2.1 find black things

The reference image is formed by black triangles. The first part is to determine with a threshold where is black. We 
take only the pixels which color has a norm below the threshold. So we get a grid, with ones where the pixel is almost 
black, and zeros otherwise.

#### 2.2.2 Find contour

A lot of algorithms could be used to do that.  
A first one is with convolution matrices. We take the matrix  
(0  1  0)  
(1 -1  1)  
(0  1  0)  
The convolution gives us numbers above one if they are in the contour, and 0 or less if they are not.   
We have to be careful because this solution take a lot of time.
If we want to find something better, this page look pretty interesting: https://www.imageprocessingplace.com/downloads_V3/root_downloads/tutorials/contour_tracing_Abeer_George_Ghuneim/alg.html

#### 2.2.3 Contours to polygons

To transform the contours in polygon, we use the Douglas-Peucker algorithm.  
We take two random points in the contour (A and B). Then, we search the point the most far to the line (C). If C is 
below a threshold, we remove all the points between A and B. If it is above, we call recursively the algorithm on A and 
C, and on C and B.

#### 2.2.4 Find the reference

Now, we have a list of polygons. But how to find the reference ?
We chose the reference to simplify as possible this part.     
The triangles have a really useful property. Once projected they remain triangles. So, we can start by removing all the 
polygons with more than 3 vertices.  
All pairs of triangles in the reference have a segment aligned. So we can search that.
Normally we shall find the four triangles doing so.

#### 2.2.5 Find the camera orientation

This is the hard part.  
Here some leads on where to look:
- homography: https://en.wikipedia.org/wiki/Homography_(computer_vision)
- https://nulldog.com/camera-pose-estimation-from-homography-of-4-points

Here a quick explanation on how it works:  
We have the following equation 
$$ \lambda PT \vec x= \vec p $$
$P$ the projection matrix \\
$T$ the camera transformation matrix \\
$\vec x$ the point in 3D space \\
$\lambda$ the zooming factor \\
$\vec p$ the position on the image (in 2D space) \\

If we develop it, we get this equation
$$\lambda 
\begin{pmatrix} 
f & 0 & C_x \\ 
0 & f & C_y \\ 
0 & 0 & 1 
\end{pmatrix}
\begin{pmatrix} 
r_{1,1} & r_{2,1} & r_{3,1} & t_x \\ 
r_{1,2} & r_{2,2} & r_{3,2} & t_y \\ 
r_{1,3} & r_{2,3} & r_{3,4} & t_z
\end{pmatrix}
\begin{pmatrix} 
x \\ y \\ z \\ 1 
\end{pmatrix}
=
\begin{pmatrix} 
x_s \\ y_s \\ s  
\end{pmatrix} $$

To simplify the problem, we take the spatial coordinates so that z = 0. So, we get the following equation

$$\lambda 
\begin{pmatrix} 
f & 0 & C_x \\ 
0 & f & C_y \\ 
0 & 0 & 1 
\end{pmatrix}
\begin{pmatrix} 
r_{1,1} & r_{2,1} & t_x \\ 
r_{1,2} & r_{2,2} & t_y \\ 
r_{1,3} & r_{2,3} & t_z
\end{pmatrix}
\begin{pmatrix} 
x \\ y \\ 1 
\end{pmatrix}
=
\begin{pmatrix} 
x_s \\ y_s \\ s  
\end{pmatrix} $$

Let $H$ be $\lambda P T$. $H$ is a square 3x3 matrix ! With the homography method, we can compute this $H$ matrix (called the homography matrix).

This matrix is all we need. In fact, we can use it to project the voxels into an image.

Note that we don't have $r_{3,1}$, $r_{3,2}$ and $r_{3,3}$. It is possibly something pretty problematic, I don't know now, we will see.

