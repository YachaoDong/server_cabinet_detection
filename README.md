# 一、简介
 机房元宇宙中有一项需要识别出机房中的服务器，由于机房道路面积狭窄，AR或其他终端手持设备视野不足，所以机房服务器在图像中呈现的区域并非矩形，而且服务器都是排列出现的，如果使用矩形框进行检测，则会出现矩形框过大，所以需要任意四边形进行检测，更加贴合目标。
 使用语义分割方法速度过慢，经过综合考量使用yolov5检测，对yolov5增加8个回归预测值。
 
 # 二、尝试方法
 ## 1. 直接预测8个坐标值（poly），不借助使用box
 直接将yolo网络输出改为8个预测值，并使用poly_iou计算。
 **遇到的问题：** 数据增广部分要改为poly，loss部分的poly_iou计算以及可导问题，NMS中的poly_iou复杂且训练速度慢，边界问题，刚训练的loss不稳定问题（由于直接回归8个0~1的预测值，不稳定）， mAP等指标评估问题。
 **结果：** 一般。

 ## 2. 预测8个值（poly）相对于poly中心点的相对距离位置
 和回归预测box wh一样，预测poly相对于 poly_xy（中心坐标）的相对距离，使用yolov5中预测wh的方法一样（预测相对于anchor_wh的比例），只不过是anchor_wh/2。
 **遇到的问题：** poly 相对于 poly_xy的距离有正有负，方向不一定，由于数据增强以及各种更加“畸形”的任意四边形，可能导致训练中的loss不能一一对应。并且计算loss时，从相对距离变化到坐标距离时容易出错。
  **结果：** 一般。

## 3. 借助box，直接预测8个值（预测8个值（poly）相对于poly中心点的相对距离位置）
将标注的poly坐标同时转换为最小外接矩形box，在使用常规yolov5 box训练的基础上增加poly 的8个值的预测。
**遇到的问题：** 训练过程中loss容易漂移，不稳定。
 **结果：** box有效果，poly效果还是差。


## 4. 借助box，预测8个到box左上角坐标的相对距离（并根据box_wh归一化后的距离），并优化细节
因为box比较稳定，有效，更改代码的过程中也容易方便，使用poly生成对应的最小外接矩形box，box按照原来yolov5的训练，包括计算NMS、计算mAP等评估结果均使用box，在此基础上增加 poly相对于box左上角的相对距离，并将其相对于box_wh进行归一化，以此来进行loss训练。
**优化细节：**
- 数据增强方面：边界处理、数据增强后对poly（labels）进行顺序排列，左上角为第一个顶点，顺时针排序，和taget_poly对应起来，为了loss对应起来。
- 训练： 因为poly严重依赖对应box的准确性，所以刚开始训练时，应该先让网络先训练box几轮，然后再增加poly_loss。和原yolov5处理box  x_c ，y_c一样，设置一个可溢出当前box的最大比例poly_out，如sigmoid后转换成 -0.5~1.5范围
 **结果：** 效果不错。
 **不足之处：** 严重依赖预测的box的准确性，且评估方法使用的是按照box进行评估。
 **TODO：**
    - [ ] 增加可导的 poly_iou进行loss 
      
    - [ ] 由于数据集目标的分布不存在重叠，可使用anchor free的目标检测方法。
    - [ ] 使用mask信息进行监督或使用速度较快的语义分割方法。  
   
 
## 5. 难点总结
1. 数据增强部分的改写，是否超出边界等，需要进行单元测试。
2. loss部分的设计。
3. iou、NMS部分的计算。
4. 评估方法的改写，画结果图。
5. 各个阶段的数据维度。

# 三、 Reference
https://github.com/ultralytics/yolov5  
https://github.com/Rhine-AI-Lab/Yolo-ArbV2
