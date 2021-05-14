# General description or introduction of the problem and your solution

In my midterm project I implemented sharpen filter in two data reuse ways and in roll and unroll version. The conclusion is obvious unroll veriosn with data resuse can have the best performance.

# Implementation details (data structure, flows and algorithms)
## data reuse
![image](https://user-images.githubusercontent.com/76727373/118217580-58aa6700-b4a8-11eb-87db-3c70cd991f0f.png)
![image](https://user-images.githubusercontent.com/76727373/118217601-6233cf00-b4a8-11eb-8cfa-9f909694cd8d.png)

My data reuse method has two phases, cold start and after cold start, cold start implies that when the buffer is empty we have to transfer three cols of data to buffer up, and after cold start means we can transfer one cols of data every time and reuse the data in buffer to reduce the num of times of transmisitions.
## not data reuse

It is the original methods that we transfer three cols of data every time and clear buffer every time.

## roll version

![image](https://user-images.githubusercontent.com/76727373/118218013-2cdbb100-b4a9-11eb-899f-021480b6ebfe.png)

In roll version I use for loop and index to make sure only one calculation will be performed at a time.

## unroll version

![image](https://user-images.githubusercontent.com/76727373/118218126-63b1c700-b4a9-11eb-8dbb-41286117d059.png)

Unroll version is obvious that we can just open up the for loop and exectuate it at the same time.


# Result

## data transfer time

![image](https://user-images.githubusercontent.com/76727373/118218273-b4c1bb00-b4a9-11eb-86c5-3504c7c9fcd2.png)

The table above shows the data transfer time. We can know that data reuse method has lower transfer count and transfer time in after cold start stage.

# simulation result

![image](https://user-images.githubusercontent.com/76727373/118218447-feaaa100-b4a9-11eb-985c-5c9f8f74535f.png)

The above table shows that data resuse and unroll version have best performance in time, which is met with our assumption. But data reuse veriosn tends to have larger area consumption, since we have to add a lot of buffer to store the data in kernel.

# Discussion
We discover that making calculation process unroll can reduce the cycle counts and maintain a reasonable area usage. Implementing data reuse will increase area usage dramatically but decrease cycle counts by half.

