---
layout: post
title:  "How to Enable DuckDB/Smallpond to Use High-Performance DeepSeek 3FS"
date:   2025-05-16 15:00:00 +0800
---


![duckdb_and_deepseek]({{ site.url }}/assets/imgs/20250516-duckdb-annd-3fs.png)

## Preface


In late February, DeepSeek open-sourced the AI storage 3FS and the data processing framework Smallpond. Smallpond is built on DuckDB and 3FS.

DeepSeek said that when a 3FS storage cluster with 25 storage nodes (using 2x400Gb networks) is set up and a GraySort test is run with Smallpond, 50 compute nodes (using 1x200Gb networks) can sort 110.5 TiB of data in 30 minutes and 14 seconds, with an average throughput of 3.66 TiB/minute. This is because 3FS's high-performance user-space interface (hf3fs_usrbio) is fully utilized.

However, DeepSeek's open-sourced Smallpond and the open-sourced DuckDB don't support accessing 3FS via the hf3fs_usrbio interface.

To boost the 3FS open-source ecosystem, the Open3FS community developed and open-sourced the DuckDB-3FS plugin, allowing DuckDB to access 3FS using the hf3fs_usrbio interface. The code repository is at <https://github.com/open3fs/duckdb-3fs> .

We also made minor changes to the Smallpond code so that when using the DuckDB engine, it can access 3FS using the hf3fs_usrbio interface. The code repository is at <https://github.com/open3fs/smallpond-3fs> .


## Download and compile the duckdb-3fs plugin


Execute the following commands on the Ubuntu 22.04 server.

```
git clone --recurse-submodules https://github.com/open3fs/duckdb-3fs
cd duckdb--3fs
GEN=ninja make -j$(nproc) BUILD_PYTHON=1 CORE_EXTENSIONS="httpfs"
```

The final generated binary files include:
```
/root/duckdb-3fs/build/release/duckdb
/root/duckdb-3fs/build/release/extension/threefs/threefs.duckdb_extension
```

## Create a 3FS cluster

Use m3fs to create a 3FS cluster, You can refer to this article <https://blog.open3fs.com/2025/03/28/deploy-3fs-with-m3fs.html> . The hostMountpoint parameter needs to be set to /3fs .

![cluster_yml]({{ site.url }}/assets/imgs/20250516-cluster-yml.png)

After the deployment is completed, check the status of the 3FS storage cluster and the mount path.

![check]({{ site.url }}/assets/imgs/20250516-check-3fs.png)


## How to Use 3FS in DuckDB

The duckdb-3fs plugin supports two 3FS path formats:
- 3fs://3fs/aaa/bbb
- /3fs/aaa/bbb  # This method requires the /3fs directory to be a mount point

Download one Parquet file to 3FS storage.
```
mkdir /3fs/duckdb/
wget https://duckdb.org/data/prices.parquet -O /3fs/duckdb/prices.parquet
```

Start the DuckDB service.
```
/root/duckdb-3fs/build/release/duckdb  --unsigned
```

Execute the following commands

```
INSTALL '/root/duckdb-3fs/build/release/extension/threefs/threefs.duckdb_extension';
LOAD threefs;
SET threefs_cluster='open3fs';
SET threefs_mount_root='/3fs/';
SET threefs_use_usrbio=true;
SET threefs_iov_size=16384;
SELECT * FROM read_parquet('3fs://3fs/duckdb/prices.parquet');
SELECT * FROM read_parquet('/3fs/duckdb/prices.parquet');
```


The output results are as follows.

![duckdb_3fs_output]({{ site.url }}/assets/imgs/20250516-duckdb-3fs-ouput.png)

If debugging is required, you can set threefs_enable_debug_logging=true.


## Install Python DuckDB

Execute the following commands.
```
cd /root/duckdb-3fs/duckdb/tools/pythonpkg
python3 -m pip install .
```

## How to Use 3FS in Python DuckDB

Use Python3 to execute the following script.
```
import duckdb

con = duckdb.connect(config={'allow_unsigned_extensions': True})

con.install_extension('/root/duckdb-3fs/build/release/extension/threefs/threefs.duckdb_extension')

con.load_extension('threefs')

con.execute("SET threefs_cluster='open3fs';")
con.execute("SET threefs_mount_root='/3fs/';")
con.execute("SET threefs_use_usrbio=true;")
con.execute("SET threefs_iov_size=16384;")
con.execute("SET threefs_enable_debug_logging=false;")

data = con.sql("SELECT * FROM read_parquet('/3fs/duckdb/prices.parquet')")
print(data)
con.sql("COPY (SELECT * FROM read_parquet('/3fs/duckdb/prices.parquet')) TO '/3fs/duckdb/output.parquet' (FORMAT PARQUET);")
data = con.sql("SELECT * FROM read_parquet('3fs://3fs/duckdb/prices.parquet')")
print(data)
```

## Download and Install Smallpond

Execute the following commands.
```
git clone https://github.com/open3fs/smallpond-3fs
cd smallpond-3fs
python3 -m build
pip3 uninstall smallpond -y
pip3 install dist/smallpond-0.15.0-py3-none-any.whl
```

## How to use 3FS in Smallpond

Note: The path format must use a path prefixed with /3fs/, and the 3FS storage mount point is the /3fs/ directory.

```
import smallpond

sp = smallpond.init(data_root="/3fs/duckdb/data_root/")
df = sp.read_parquet("/3fs/duckdb/prices.parquet")
df = df.repartition(3, hash_by="ticker")
df = sp.partial_sql("SELECT ticker, min(price), max(price) FROM {0} GROUP BY ticker", df)
print(df.to_pandas())
df.write_parquet("/3fs/duckdb/output/")
```


![smallpond_output]({{ site.url }}/assets/imgs/20250516-smallpond-output.png)

Smallpond can directly use the path format of /3fs/aaa/bbb. The duckdb engine will automatically recognize that this path refers to 3FS storage and then automatically use the high-performance hf3fs_usrbio interface to access 3FS files.

The arrow engine in Smallpond uses the standard POSIX interface for reading and writing, so there's no need to modify the arrow code.


## Summary

With the Duckdb-3FS plugin, we can enable DuckDB to use the high-performance interface hf3fs_usrbio to access the 3FS storage cluster.

It is hoped that the functionality of this plugin will allow the DuckDB ecosystem to fully leverage the storage performance of 3FS.

If you are interested in these two projects, you are welcome to submit issues and pull requests.
- <https://github.com/open3fs/duckdb-3fs>
- <https://github.com/open3fs/smallpond-3fs>
