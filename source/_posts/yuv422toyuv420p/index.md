---
title: yuv422转换为yuv420p
date: 2018-06-12 22:25:24
---


今天遇到一个问题，我的摄像头采集到的数据是yuyv格式(属于yuv422)，而X264在进行编码的时候需要标准的YUV（4：2：0）。所以有一个yuv422toyuv420的转换。在网上找了半天找到的方法拿过来转换了查看都很花。于是自己看了一下yuv格式的解释，准备写一个转换代码。以下许多解释都是按我的理解：

# 1.yuv
yuv格式通常有两大类：打包(packed)和平面(planar)格式。前者在码流里是yuv挨一起，比如我的yuyv就是 Y0 U0 Y1 V1 Y2 U2 …. 每一个 Y对应一组UV分量。后者存储y u v分量是分开存储的，这种方式一般后面带P， 比如y uv420p就是Y0 Y1 Y2 … U0 U1 U2 … V0 V1 V2 … uv分量的多少根据格式来，yuv420也就是每四个 Y共用一组UV分量。

# 2.转换
理解了yuyv即yuv422与yuv420p中分量的排布，就要进行转换了。网上查到的资料说yuv422->yuv420p时 丢弃偶数行的uv分量。

# 3.编码

    定义：

    unsignedchar *y = out;
    unsignedchar *u = out + width*height;
    unsigned char*v = out + width*height + width*height/4;


y u v分别指向yuv420buf中存储y u v分量的数组，这里out的类型为char型数组，按yuv420p的定义，4个y共用一对uv，那么一个y对应1/4个uv，一个分量占一个byte，out的大小为：总共的y分量(width*height) + 总共的u分量(width*height/4) + 总共的v分量(width*height/4) = width*height*3/2。通过上面的转换也可以得到yuyv(yuv422)一个像素占用2个字节，yuv420p一个像素占1.5个字节，rgb24的话占用3个字节，还是节约了一点点空间的。。。

    获取y分量并存储到yuv420buf中：
             for(i=0; i<yuv422_length; i+=2){
                   *(y+y_index) = *(in+i);
                   y_index++;
             }
    这里的yuv422_length为width*height*2;y_index初始为0，存储一个y就自加一次。

    获取uv分量并存储到yuv420buf中：
             for(i=0; i<height; i+=2){
                   base_h = i*width*2;
                   for(j=base_h+1;j<base_h+width*2; j+=2){
                            if(is_u){
                                     *(u+u_index)= *(in+j);
                                     u_index++;
                                     is_u = 0;
                            }
                            else{
                                     *(v+v_index)= *(in+j);
                                     v_index++;
                                     is_u = 1;
                            }
                   }
              }

*总结：初入视频图像，我还是一个菜鸟，对于很多理解也不深，这个代码应该还有很多没考虑，对于我可用了。当然以上都是废话，直接贴代码*

    int yuv422toyuv420(unsigned char *out, const unsigned char *in, unsigned int width, unsigned int height)
        {
        unsigned char *y = out;
        unsigned char *u = out + width*height;
        unsigned char *v = out + width*height + width*height/4;

        unsigned int i,j;
        unsigned int base_h;
        unsigned int is_y = 1, is_u = 1;
        unsigned int y_index = 0, u_index = 0, v_index = 0;

        unsigned long yuv422_length = 2 * width * height;

        //序列为YU YV YU YV，一个yuv422帧的长度 width * height * 2 个字节
        //丢弃偶数行 u v

        for(i=0; i<yuv422_length; i+=2){
            *(y+y_index) = *(in+i);
            y_index++;
        }

        for(i=0; i<height; i+=2){
            base_h = i*width*2;
            for(j=base_h+1; j<base_h+width*2; j+=2){
                if(is_u){
                    *(u+u_index) = *(in+j);
                    u_index++;
                    is_u = 0;
                }
                else{
                    *(v+v_index) = *(in+j);
                    v_index++;
                    is_u = 1;
                }
            }
        }

        return 1;
        }
