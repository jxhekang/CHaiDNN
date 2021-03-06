## **CHaiDNN API Details**

There are 3 API calls:

1. Initialization 
2. Execute 
3. Release

### **Initialization API**
--------------------------------------------------------------
```c++
void xiInit(char *dirpath, 
char* prototxt, 
char* caffemodel, char* meanfile, 
int mean_type, std::vector<xChangeLayer> (&jobQueue)[NUM_IMG], 
int resize_h, int resize_w, bufPtrs ptrsList);	
```
>`dirpath`		: Directory Path of the Network. Keep caffe all files  in this directory (.prototxt, .caffemodel, mean file).
			  Please create a folder with name "data" inside this directory as the weights are generated inside this.


>`prototxt`	: Name of the prototxt file existing inside "dirpath". <E.g. deploy.prototxt>

>`caffemodel`	: Name of the caffemodel file existing inside "dirpath". <E.g. SSD.caffemodel>

>`meanfile`	: Name of the mean file existing inside "dirpath". <E.g. mean_file.txt>

>`mean_type`	: Type of mean file. 0 For .txt file & 1 for .binaryproto file.

>`jobQueue`	: An array of vector "xChangeLayer". Required for initialization and execution. See example for more details.

>`ptrsList`	: Struct for buffer handles. Required for releasing memory after execution. See example for more details. 

>`resize_h`	: Resized height for input image (Mostly the input dimesions mentioned in prototxt)

>`resize_w`	: Resized width for input image (Mostly the input dimesions mentioned in prototxt)

### **Execution API**
--------------------------------------------------------------
```c++
unsigned short xiExec(std::vector<xChangeLayer> jobQueue[NUM_IMG], 
void *inptr, 
void *outptr, 
int inBytes);  
```           
> `jobQueue`	: Array of vector initialized in xiInit Function.

> `inptr`		: Input Buffer. Should contain input image data/video frame resized to network input size.
			  Input should be 3 channel in BGR (pixel packed) format where B is LS-Byte and R is MS-Byte.

> `outptr`		: Output Buffer. Create memory for output data. This will be filled with output of the network inside 
			  xiExec function.

> `inBytes`		: Number of bytes for input pixels to the API. This will be 1 if the (input-mean) data is 8-bit and 2 when 
			  (input-mean) is 16-bits.

### Input Data
--------------------------------------------------------------
Input should be 3 channel in BGR (pixel packed) format where B is LS-Byte and R is MS-Byte. 
> B[7:0], G[15:8], R[23:16]


### Output Data:
--------------------------------------------------------------------
Output of the network is written into the output buffers (outptr1 and outptr2). For different networks, the o/p data
organization changes based on the last layer in the network. For now, we are supporting 3 different kinds of data 
organization for o/p buffers.

1. Classification Network

	For these kind of networks, the Softmax layer would be the last layer. Data organization for these kind of networks will be as follows. The output buffers will contain probability values for the number of classes. The index values will always start 
	  from 0.

	>Example: For GoogleNet, the output is probabilities of 1000 Classes. Position of the probability value gives the 
   class ID (Starts from 0).
   (List of Class IDs can be found [here](https://gist.github.com/yrevar/942d3a0ac09ec9e5eb3a))   

2. Detection Networks

	For these kind of networks, the last layer will most probably be NMS later. The output of the NMS layer is box ID, class ID, score (probability), co-ordinates.

	>How to Interpret output data for Detection networks :
	First entry of the output buffer is number of output boxes generated by the network for an Image.
	```c++
       Example:
	   int nOutBoxes1 = ((int*)outptr1)[0];
	   int nOutBoxes2 = ((int*)outptr2)[0];
	```
	>From second entry onwards the output is arranged in the below format. Each Box will have the following 7 values.
	```
	Box-ID, 
	ClassID, 
	Score(Prob), 
	Coordinates(Xmin, Ymin, Xmax, Ymax)

	This order will be followed for all the output boxes.
	```

	>With number of output boxes, user can read all the output boxes generated by SSD.
	
	>Xmin, Ymin, Xmax, Ymax are floating point values. To get the correct pixel positions or co-ordinates of the boxes 
	   in the image, user has to multiply these values with input height/width (in case of SSD : 300 x 300).

3. Segmentation Networks

	For these kind of networks, the output layer is Crop and the size of the output layer will be same as the input size of the network.
	
	The output will be written sequentially in raster scan order inside output buffers. 
	
	>Example: For AlexNet-FCN, the last layer is Crop and the output size is 500 x 500.
  
### **Release API**
--------------------------------------------------------------
```c++
void xiRelease(bufPtrs ptrsList);
```
>`ptrsList`	: ptrsList initialized in xiInit Function. Required to release memory.