---title--- binary image compression with python and numpy 
---abstract--- numpy-thonic run length compression 

I've been tinkering with my [cam_board](https://github.com/kacpertopol/cam_board) script.
This little program can turn a web cam into a white / black board. I found it handy these rough, 
socially isolated, disease ridden times. The de-noising feature of the script is particularly
useful when recording the "blackboard" for students because it greatly reduces the video file sizes.
I was wondering if it would be possible to have a compression algorithm built into the script. 
Ideally, to reduce compression time, the compression procedure would rely heavily on the fast methods of the `numpy` library.
This post describes the resulting algorithm. The description is also part of an assignment where
I ask students to implement the same algorithm without `numpy` and compare compression times.
The script and sample image can be downloaded from [here](https://github.com/kacpertopol/rcnumpy).

