# Tensorflow Java binding

This example shows how to use a trained model in SavedModel format in Java to detect classes, scores and objects in an image. More info at: https://github.com/tensorflow/models/tree/master/samples/languages/java/object_detection

1. Create a Java project with Maven or Gradle
    1. For Maven add in pom.xml
    ```
    <dependency> 
    <groupId>org.tensorflow</groupId> 
    <artifactId>tensorflow</artifactId> 
    <version>1.5.0</version> 
    </dependency>
    ```
    2. For Gradle add in build.gradle
    ```
    compile group: 'org.tensorflow', name: 'tensorflow', version: '1.5.0'
    ```
2. Create a class in src/main/java with the name 'DetectObjects' with the content from the following link: https://github.com/tensorflow/models/blob/master/samples/languages/java/object_detection/src/main/java/DetectObjects.java

3. In src/main/java/object_detection/protos create a class with the name 'StringIntLabelMapOuterClass' with the content from the following link: https://github.com/tensorflow/models/blob/master/samples/languages/java/object_detection/src/main/java/object_detection/protos/StringIntLabelMapOuterClass.java
4. In DetectObjects:
    1. args[0] = path to saved_model from any model (eg. "ssd_mobilenet_v1_coco_2017_11_17\\saved_model")
    2. args[1] = path to labels (eg. "labels\\mscoco_label_map.pbtxt")
    3. args[2] = path to an image (eg. "images\\image11.jpg")
5. When runned, the code will return classes and scores found in the image. Boxes must be printed manually and are stored in a variable names 'boxes'.

