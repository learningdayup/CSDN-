﻿---
Tensorflow-keras实战（八）：tfrecords实战
---


# 目录：

1.tfrecord基础
2.tfrecords实战



### 1.tfrecord基础
```
import matplotlib as mpl
import matplotlib.pyplot as plt
%matplotlib inline
import numpy as np
import pandas as pd
import os
import sklearn
import sys
import time
import tensorflow as tf

from tensorflow import keras

print(tf.__version__)
print(sys.version_info)
for module in mpl,np,pd,sklearn,tf,keras:
    print(module.__name__, module.__version__)



# tfrecord 文件格式
# -> tf.train.Example
#      ->tf.train.Features->{"key":tf.train.Feature}
#         -> tf.train.Feature -> tf.train.ByteList/FloatList/Int64List


favorite_books = [name.encode('utf-8') for name in ["machine learning","cc150"]]

favorite_books_bytelist = tf.train.BytesList(value = favorite_books)
print(favorite_books)

hours_floatlist = tf.train.FloatList(value = [15.5,9.5,7.0,8.0])
print(hours_floatlist)

age_int64list = tf.train.Int64List(value = [42])
print(age_int64list)

features = tf.train.Features(
    feature = {
        "favorite_books":tf.train.Feature(bytes_list=favorite_books_bytelist),
        "hours":tf.train.Feature(float_list = hours_floatlist),
        "age":tf.train.Feature(int64_list = age_int64list),
    }
)

print(features)



example = tf.train.Example(features = features)
print(example)

serialized_example = example.SerializeToString()
print(serialized_example)



output_dir = 'tfrecord_basic'
if not os.path.exists(output_dir):
    os.mkdir(output_dir)
filename = "test.tfrecord"
filename_fullpath = os.path.join(output_dir,filename)
with tf.io.TFRecordWriter(filename_fullpath) as writer:
    for i in range(3):
        writer.write(serialized_example)



dataset = tf.data.TFRecordDataset([filename_fullpath])
for serialized_example_tensor in dataset:
    print(serialized_example_tensor)



expected_features = {
    "favorite_books":tf.io.VarLenFeature(dtype = tf.string),
    'hours':tf.io.VarLenFeature(dtype = tf.float32),
    "age":tf.io.FixedLenFeature([],dtype = tf.int64),
}
dataset = tf.data.TFRecordDataset([filename_fullpath])
for serialized_example_tensor in dataset:
    example = tf.io.parse_single_example(
        serialized_example_tensor,
        expected_features)
    books = tf.sparse.to_dense(example["favorite_books"],default_value = b"")
    for book in books:
        print(book.numpy().decode("UTF-8"))




#压缩形式存储和读取
filename_fullpath_zip = filename_fullpath + '.zip'
options = tf.io.TFRecordOptions(compression_type="GZIP")
with tf.io.TFRecordWriter(filename_fullpath_zip,options) as writer:
    for i in range(3):
        writer.write(serialized_example)



dataset_zip = tf.data.TFRecordDataset([filename_fullpath_zip],compression_type="GZIP")
for serialized_example_tensor in dataset_zip:
    example = tf.io.parse_single_example(
        serialized_example_tensor,
        expected_features)
    books = tf.sparse.to_dense(example["favorite_books"],default_value = b"")
    for book in books:
        print(book.numpy().decode("UTF-8"))


```




### 2.tfrecords实战


```
import matplotlib as mpl
import matplotlib.pyplot as plt
%matplotlib inline
import numpy as np
import pandas as pd
import os
import sklearn
import sys
import time
import tensorflow as tf

from tensorflow import keras

print(tf.__version__)
print(sys.version_info)
for module in mpl,np,pd,sklearn,tf,keras:
    print(module.__name__, module.__version__)


source_dir = "./generate_csv/"

def get_filenames_by_prefix(source_dir,prefix_name):
    all_files = os.listdir(source_dir)
    results = []
    for filename in all_files:
        if filename.startswith(prefix_name):
            results.append(os.path.join(source_dir,filename))
    return results

train_filenames = get_filenames_by_prefix(source_dir,"train")
valid_filenames = get_filenames_by_prefix(source_dir,"valid")
test_filenames = get_filenames_by_prefix(source_dir,"test")

import pprint
pprint.pprint(train_filenames)
pprint.pprint(valid_filenames)
pprint.pprint(test_filenames)


def parse_csv_line(line,n_fields=9):
    defs = [tf.constant(np.nan)]*n_fields
    parsed_fields = tf.io.decode_csv(line,record_defaults=defs)
    x = tf.stack(parsed_fields[0:-1])
    y = tf.stack(parsed_fields[-1:])
    return x,y

def csv_reader_dataset(filenames,n_readers=5,batch_size=32,n_parse_threads=5,
                      shuffle_buffer_size=10000):
    dataset = tf.data.Dataset.list_files(filenames)
    dataset = dataset.repeat()
    dataset = dataset.interleave(
        lambda filename:tf.data.TextLineDataset(filename).skip(1),
        cycle_length = n_readers)
    dataset.shuffle(shuffle_buffer_size)
    dataset = dataset.map(parse_csv_line,
                          num_parallel_calls = n_parse_threads)
    dataset = dataset.batch(batc    x = tf.stack(parsed_fields[0:-1])
    y = tf.stack(parsed_fields[-1:])
    return x,y

def csv_reader_dataset(filenames,n_readers=5,batch_size=32,n_parse_threads=5,
                      shuffle_buffer_size=10000):
    dataset = tf.data.Dataset.list_files(filenames)
    dataset = dataset.repeat()
    dataset = dataset.interleave(
        lambda filename:tf.data.TextLineDataset(filename).skip(1),
        cycle_length = n_readers)
    dataset.shuffle(shuffle_buffer_size)
    dataset = dataset.map(parse_csv_line,
                          num_parallel_calls = n_parse_threads)
    dataset = dataset.batch(batch_size)
    return dataset

batch_size=32
train_set = csv_reader_dataset(train_filenames,
                              batch_size=batch_size)
valid_set = csv_reader_dataset(valid_filenames,
                             batch_size=batch_size)
test_set = csv_reader_dataset(test_filenames,
                             batch_size=batch_size)h_size)
    return dataset

batch_size=32
train_set = csv_reader_dataset(train_filenames,
                              batch_size=batch_size)
valid_set = csv_reader_dataset(valid_filenames,
                             batch_size=batch_size)
test_set = csv_reader_dataset(test_filenames,
                             batch_size=batch_size)




def serialize_example(x,y):
    """Converts x,y to tf.train.Example and serialize"""
    input_features = tf.train.FloatList(value = x)
    label = tf.train.FloatList(value = y)
    features = tf.train.Features(
        feature = {
            "input_features":tf.train.Feature(float_list = input_features),
            "label":tf.train.Feature(float_list = label)
        }
    )
    example = tf.train.Example(features = features)
    return example.SerializeToString()

def csv_dataset_to_tfrecords(base_filename,dataset,
                             n_shards,steps_per_shard,compression_type = None):
    options = tf.io.TFRecordOptions(compression_type=compression_type)
    all_filenames = []
    for shard_id in range(n_shards):
        filename_fullpath = '{}_{:05d}-of-{:05d}'.format(base_filename,shard_id,n_shards)
        with tf.io.TFRecordWriter(filename_fullpath,options) as writer:
            for x_batch,y_batch in dataset.take(steps_per_shard):
                for x_example,y_example in zip(x_batch,y_batch):
                    writer.write(serialize_example(x_example,y_example))
        all_filenames.append(filename_fullpath)
        
    return all_filenames


n_shards = 20
train_steps_per_shard = 11610//batch_size//n_shards
valid_steps_per_shard = 3880//batch_size//n_shards
test_steps_per_shard = 5170//batch_size//n_shards

output_dir = "generate_tfrecords"
if not os.path.exists(output_dir):
    os.mkdir(output_dir)
    
train_basename = os.path.join(output_dir,"train")
valid_basename = os.path.join(output_dir,"valid")
test_basename = os.path.join(output_dir,"test")

train_tfrecord_filenames = csv_dataset_to_tfrecords(
    train_basename,train_set,n_shards,train_steps_per_shard,None)
valid_tfrecord_filenames = csv_dataset_to_tfrecords(
    train_basename,valid_set,n_shards,train_steps_per_shard,None)
test_tfrecord_filenames = csv_dataset_to_tfrecords(
    train_basename,test_set,n_shards,train_steps_per_shard,None)



n_shards = 20
train_steps_per_shard = 11610//batch_size//n_shards
valid_steps_per_shard = 3880//batch_size//n_shards
test_steps_per_shard = 5170//batch_size//n_shards

output_dir = "generate_tfrecords_zip"
if not os.path.exists(output_dir):
    os.mkdir(output_dir)
    
train_basename = os.path.join(output_dir,"train")
valid_basename = os.path.join(output_dir,"valid")
test_basename = os.path.join(output_dir,"test")

train_tfrecord_filenames = csv_dataset_to_tfrecords(
    train_basename,train_set,n_shards,train_steps_per_shard,compression_type="GZIP")
valid_tfrecord_filenames = csv_dataset_to_tfrecords(
    train_basename,valid_set,n_shards,train_steps_per_shard,compression_type="GZIP")
test_tfrecord_filenames = csv_dataset_to_tfrecords(
    train_basename,test_set,n_shards,train_steps_per_shard,compression_type="GZIP")



pprint.pprint(train_tfrecord_filenames)
pprint.pprint(valid_tfrecord_filenames)
pprint.pprint(test_tfrecord_filenames)


expected_features = {
    "input_features":tf.io.FixedLenFeature([8],dtype = tf.float32),
    "label":tf.io.FixedLenFeature([1],dtype = tf.float32)
}

def parse_example(serialized_example):
    example = tf.io.parse_single_example(serialized_example,
                                        expected_features)
    return example["input_features"],example["label"]

def tfrecords_reader_dataset(filenames,n_readers=5,batch_size=32,n_parse_threads=5,
                             shuffle_buffer_size=10000):
    dataset = tf.data.Dataset.list_files(filenames)
    dataset = dataset.repeat()
    dataset = dataset.interleave(
        lambda filename:tf.data.TFRecordDataset(
            filename,compression_type = "GZIP"),
        cycle_length = n_readers)
    dataset.shuffle(shuffle_buffer_size)
    dataset = dataset.map(parse_example,
                          num_parallel_calls = n_parse_threads)
    dataset = dataset.batch(batch_size)
    return dataset

tfrecords_train = tfrecords_reader_dataset(train_tfrecord_filenames,batch_size=3)

for x_batch,y_batch in tfrecords_train.take(2):
    print(x_batch)
    print(y_batch)



batch_size = 32
tfrecords_train_set = tfrecords_reader_dataset(
    train_tfrecord_filenames,batch_size=batch_size)
tfrecords_valid_set = tfrecords_reader_dataset(
    valid_tfrecord_filenames,batch_size=batch_size)
tfrecords_test_set = tfrecords_reader_dataset(
    test_tfrecord_filenames,batch_size=batch_size)



model = keras.models.Sequential([
    keras.layers.Dense(30,activation="relu",
                      input_shape=[8]),
    keras.layers.Dense(1),
])

model.compile(loss="mean_squared_error",optimizer="sgd")
callbacks = [keras.callbacks.EarlyStopping(
    patience=5,min_delta=1e-2)]

history = model.fit(tfrecords_train_set,
                    validation_data = tfrecords_valid_set,
                    steps_per_epoch = 11160//batch_size,
                    validation_steps = 3870//batch_size,
                    epochs = 100,
                    callbacks = callbacks)


model.evaluate(tfrecords_test_set,steps = 5160//batch_size)

```











