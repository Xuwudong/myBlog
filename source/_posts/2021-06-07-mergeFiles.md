---
title: 堆的应用-合并有序小文件
tags:
  - 数据结构与算法
categories:
  - 数据结构与算法
date: 2021-06-07 21:05:35
---

&ensp;&ensp;假设我们有 100 个小文件，每个文件的大小是 100MB，每个文件中存储的都是有序的字符串。我们希望将这些 100 个小文件合并成一个有序的大文件。这里就会用到优先级队列。

&ensp;&ensp;整体思路有点像归并排序中的合并函数。我们从这 100 个文件中，各取第一个字符串，放入数组中，然后比较大小，把最小的那个字符串放入合并后的大文件中，并从数组中删除。

&ensp;&ensp;假设，这个最小的字符串来自于 13.txt 这个小文件，我们就再从这个小文件取下一个字符串，并且放到数组中，重新比较大小，并且选择最小的放入合并后的大文件，并且将它从数组中删除。依次类推，直到所有的文件中的数据都放入到大文件为止。

&ensp;&ensp;这里我们用数组这种数据结构，来存储从小文件中取出来的字符串。每次从数组中取最小字符串，都需要循环遍历整个数组，显然，这不是很高效。有没有更加高效方法呢？

&ensp;&ensp;我们知道，删除堆顶数据和往堆中插入数据的时间复杂度都是 O(logn)，n 表示堆中的数据个数，这里就是 100。是不是比原来数组存储的方式高效了很多呢？

&ensp;&ensp;思路已经有了，这里我用Java实现了一个小规模的合并100个包含有序数字的demo:

1.声明几个静态变量
``` bash
/** 有序文件列表**/
public static ArrayList<File> fileList = new ArrayList<>();    
 /** 小顶堆 **/
private static Heap<Node> heap = new Heap<>(101, new Comparator<Node>() {        
    @Override
    public int compare(Node o1, Node o2) {            
       if (Long.parseLong(o1.getWord()) > Long.parseLong(o2.getWord())) {               
           return -1;
       } else if (Long.parseLong(o1.getWord()) < Long.parseLong(o2.getWord())) {                
           return 1;
       } else {               
           return 0;
       }
   }
});   
 /**  file -> fileReader 保存每个文件的输入流**/
private static HashMap<File, BufferedReader> readerMap = new HashMap<>();    
/** 每个文件的行数 **/
private static int fileLine = 100000;
```

2.创建小文件并写入内容
```bash
/**
 * 创建小文件(如果存在，将其删除）并写入内容
 *
 * @param dic
 */
public static void create(String dic) throws IOException {
    File file = new File(dic);        
    if (!file.exists()) {
        file.mkdirs();
    }        
    if (file.isDirectory()) {
        String fileName = dic + "\\test";            
        for (int i = 0; i < 100; i++) {
           File f = new File(fileName + i + ".txt");                if (!f.exists()) {
               f.createNewFile();
               writeFileContent(f.getAbsolutePath(), i);
           } else {
                f.delete();
                f.createNewFile();
                writeFileContent(f.getAbsolutePath(), i);
            }
        }
    }
}    
/**
 * 写入内容，每行相差<!--fileLine-->
 *
 * @param fileName
 * @param i
 */
public static void writeFileContent(String fileName, long i) {
    File file = new File(fileName);        
    if (file.exists()) {            
        try (PrintWriter pw = new PrintWriter(new FileOutputStream(file))) {                
            for (int j = 0; j < fileLine; j++) {
                pw.write(i + "\r\n");
                pw.flush();
                i = i + fileLine;
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```
3.初始化fileList和fileReaded
```bash
public static void init(File file) throws FileNotFoundException {        
    if (file.isDirectory()) {
        File[] files = file.listFiles();            
        for (File f : files) {                
            if (f.isDirectory()) {
                init(f);
            } else {                    
                if (f.getName().indexOf("bigFile") < 0) {
                    fileList.add(f);
                    readerMap.put(f, new BufferedReader(new InputStreamReader(new FileInputStream(f))));
                }
            }
        }
    } else {            
        if (file.getName().indexOf("bigFile") < 0) {
            fileList.add(file);
            readerMap.put(file, new BufferedReader(new InputStreamReader(new FileInputStream(file))));
        }
    }
}
```
4.创建大文件
```bash
/**
 * 新建大文件，如果存在，将其删除
 * 
 * @return
 * @throws IOException
 */
public static File createBigFile() throws IOException {
   File bigFile = new File("D:\\data\\mergeFileData\\bigFile.txt");        
   if (!bigFile.exists()) {
        bigFile.createNewFile();
   }        
   try (FileWriter fileWriter = new FileWriter(bigFile)) {
        fileWriter.write("");
        fileWriter.flush();
   }        
   return bigFile;
}
```
5.初始化堆，从每个小文件读入一行到堆中。
```bash
public static void initHeap(File bigFile) throws FileNotFoundException {        
    for (File file : fileList) {
         BufferedReader reader = readerMap.get(file);
         String word;            
         try {
            word = reader.readLine();                
            if (word != null) {
                heap.insert(new Node(word, file));
            }
       } catch (IOException e) {
           e.printStackTrace();
       }
    }
}
```
6.循环堆，删除堆顶元素，并将堆顶元素写入大文件中，若堆顶元素来自于此时被读的文件，则继续读此文件，否则继续删除堆顶元素以读取下一个文件.
```bash
public static void move(File bigFile) {        
    try (PrintWriter pw = new PrintWriter(new FileOutputStream(bigFile))) {            
        while (heap.getCount() > 0) {
            Node minNode = heap.deleteFirst();
            pw.write(minNode.getWord() + "\r\n");
            File file = minNode.getFile();
            BufferedReader reader = readerMap.get(file);
            String word = reader.readLine();                
            if (word != null && word != "\r\n") {
                Node node = new Node(word, file);
                heap.insert(node);
            }
        }
        heap.print();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```


完整代码链接（文件为MergeFiles）：

https://github.com/Xuwudong/algo