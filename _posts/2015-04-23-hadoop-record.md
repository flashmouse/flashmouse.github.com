---
layout: post
title: 公司逗比hadoop集群使用记录(陆陆续续)
categories: [java,hadoop]
tags: [记录,hadoop,java]
---

截止今天，在这个坑爹公司的惨烈数据组(修罗场)待了1年有余。说是数据组，每天的事情基本都是导报表----这玩意连报表都说不上---导数据，导数据和因为这样那样的原因重跑数据然后重导数据。虽然hadoop集群还只是1.04版本，说是集群机器数一共只有个位数……老实说真要换下家的时候，面试官亲切的问我：***你说你做过hadoop，你们的规模有多大啊？***负分滚粗！

虽然做了1年很多东西都快忘记了，姑且按照evernote里的记录，看看还能翻出多少来，装饰一下这片空白的网站……

我想想啊，其实集群不是我搭建的，集群遇到问题我基本也不去处理退给搭建集群的组长慢慢折腾，这一年和hadoop有关的操作主要集中在利用java程序去读写hdfs、hive和hbase上。所以就是写java操作hdfs的程序，一些mr代码等。姑且记录一下遇到的问题……想到多少写多少。

1. [写mapreduce项目时，如何将外部jar包导入classpath目录。](#tag1)
2. [用mr项目从hbase导入导出数据](#tag2)
3. [mr项目将数据导出到mysql时的问题。](#tag3)
4. [hive中hql直接导出数据到mysql](#tag4)

###<span id="tag1">  写mapreduce项目时，如何将外部jar包导入classpath目录。</span>
一个mr的jar包丢给hadoop执行时，hadoop会将这个包复制到所有即将运行这个jar包里map和reduce任务的task节点上。但是jar包引用的外部jar包和配置文件需要我们手动丢到hdfs的任务缓存目录上。我是参考[这篇文章](http://blog.cloudera.com/blog/2011/01/how-to-include-third-party-libraries-in-your-map-reduce-job/)的内容完成的。具体怎么操作的不记得啊- -。貌似是加了个参数-libjar还是啥的，还得写绝对路径一个文件一个文件的加。将来如果还有哪个胆大包天的公司敢让我做hadoop得话大概或许还能有机会碰上这个问题恩。

###<span id="tag2"> 用mr项目从hbase导入导出数据。</span>

说实在话，我觉得hbase简直弱爆了，count速度无法直视，查数据只能以k-v的形式进行，如果对key加上filter操作速度大幅下降，我觉得还不如我参加的mysql的开源项目mycat好用。功能比hbase多，速度还快，成本还低……

这个不错，还能翻到写过的代码……懒得写了贴代码← ←

mr类：
```java
package com.tianci.admr.mr;

import java.io.IOException;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

import org.apache.commons.io.output.ByteArrayOutputStream;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.TableReducer;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.util.ToolRunner;

import com.tianci.admr.mr.conf.tool.HBaseTool;

public class SendMac2HBase {
	public static class Map extends Mapper<LongWritable, Text, Text, Text> {
		@Override
		public void map(LongWritable key, Text value, Context context)
				throws IOException {
			String[] fields = value.toString().split("\\|");
			String mac = fields[0];
			int len = fields.length;
			if (len < 2) {
				return;
			}
			for (int i = 1; i < len; i++) {
				try {
					context.write(new Text(mac), new Text(fields[i]));
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		}
	}

	public static class Reduce extends
			TableReducer<Text, Text, ImmutableBytesWritable> {
		@Override
		public void reduce(Text key, Iterable<Text> values, Context context)
				throws IOException {
			Iterator<Text> it = values.iterator();
			Put putrow = null;
			List<String> tags = new ArrayList<String>();
			while (it.hasNext()) {
				tags.add(it.next().toString());
			}
			OutputStream bos = new ByteArrayOutputStream();
			ObjectOutputStream oos = new ObjectOutputStream(bos);
			byte[] re = null;
			try {
				oos.writeObject(tags);
				re = ((ByteArrayOutputStream) bos).toByteArray();
			} catch (Exception e) {

			} finally {
				bos.close();
				oos.close();
			}
			if (re == null) {
				return;
			}
			putrow = new Put(key.toString().getBytes());
			putrow.add("tags".getBytes(), "tags".getBytes(), re);
			try {
				context.write(new ImmutableBytesWritable(key.toString()
						.getBytes()), putrow);
			} catch (IOException e) {
				e.printStackTrace();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	/**
	 * 
	 * @param args
	 *            0:in path
	 * @throws Exception
	 */
	public static void main(String[] args) throws Exception {
		int mr;
		mr = ToolRunner.run(new Configuration(), new HBaseTool(), args);
		System.exit(mr);
	}
}
```

配置类：
```java
package com.tianci.admr.mr.conf.tool;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.filecache.DistributedCache;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.util.Tool;

import com.tianci.admr.mr.SendMac2HBase;

public class HBaseTool extends Configured implements Tool {

	@Override
	public int run(String[] arg0) throws Exception {
		Configuration conf = HBaseConfiguration.create();
		// conf.set("hbase.zookeeper.quorum.", "localhost"); // 千万别忘记配置

		Job job = new Job(conf, "mac-key-tags-Hbase");
		job.setJarByClass(SendMac2HBase.class);

		Path in = new Path(arg0[0]);

		job.setInputFormatClass(TextInputFormat.class);
		FileInputFormat.addInputPath(job, in);

		job.setMapperClass(SendMac2HBase.Map.class);
		job.setReducerClass(SendMac2HBase.Reduce.class);
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(Text.class);
		
//		DistributedCache.addArchiveToClassPath(archive, conf);

		TableMapReduceUtil.initTableReducerJob("mac2tags",
				SendMac2HBase.Reduce.class, job);

		job.waitForCompletion(true);
		return 0;
	}

}
```

这是个从hdfs读数据写入hbase的mr，从hbase读数据也差不多啦恩！不知道hadoop2.X这些类还在不在……

###<span id="tag3"> mr项目将数据导出到mysql时的问题。</span> 

mr项目将数据导出导mysql主要是用DBOutputFormat类。新版本的hadoop不知道怎么样，公司集群使用的hadoop版本中这个类很2，开了源码一看，里面用的jdbc的PreparedStatement，所有数据全部都准备完了它才commit到数据库。我这一个mr任务可是要往数据库插几百万条数据啊- -理所当然的内存吃不消跪了。可以修改源代码，让它每X条就commit一次，还有个方法是像我做得这样，手动设置这个任务的reduce数量```job.setNumReduceTasks(64);```，这样每次commit的记录数就会变少……就可以避免内存跪掉的问题……

###<span id="tag4"> hive中hql直接导出数据到mysql </span>

这个用导了hive的udf，用户自定义函数，把那个啥jar包丢到hive里，就可以……找不到了恩_(:з」∠)_