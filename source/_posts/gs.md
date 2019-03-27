---
title: 用Geoserver搭建自己的地图服务
date: 2017-09-26 17:07:12

categories:
- 技术

tags:
- geoserver
- postgis
- openStreetMap
- openLayers
---

## 安装 Geoserver

下载[geoserver][1]并解压

```
cd /geoserver/bin/
./startup.bat
```

访问[geoServer][2] 初始账号：admin  密码：geoserver

geoServer默认端口为8080，其值对应文件start.ini里的jetty.port，可修改

## Geoserver 数据源

### PostGis

ubuntu 16.04 [安装教程][3]

#### Postgresql 设置默认用户 postgres 的密码

```
// 登录到 postgresql 窗口
sudo -u postgres psql postgres

// 用下面的命令为用户 postgres 设置密码：
postgres=# \password postgres
Enter new password:
Enter it again:
postgres=# \q
```

#### Postgresql 开启远程链接

```
// 找到pg_hba.conf文件，加入
host all all 0.0.0.0/0 trust
// 重启postgresql
sudo service postgresql reload
```

#### 安装 osm2pgsql

```
cd ~
git clone git://github.com/openstreetmap/osm2pgsql.git
cmake ..
make 
sudo make install
```

也可以参考github上面的步骤，少什么依赖就装什么

#### 导入osm数据(不包含openstreetmap-carto样式)

下载[OpenStreetMap][4]

为postgresql安装以下扩展

```
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
CREATE EXTENSION ogr_fdw;
CREATE EXTENSION hstore;
```

正式导入数据

```
osm2pgsql -s -U postgres -H 127.0.0.1 -P 5432 -W -d chinaosmgisdb /tmp/china-latest.osm.pbf
```

用户，数据库，文件地址按实际情况变动即可

####  导入openstreetmap-carto样式

下载[openstreetmap-carto][5]

正式导入数据

```
su postgres

osm2pgsql -s -U postgres -H 127.0.0.1 -P 5432 -W -d chinaosmgisdb /tmp/china-latest.osm.pbf --style /home/hldev/openstreetmap-carto-master/openstreetmap-carto.style
```

###  shapefile

curl -v -u admin:@pp\$4boundleSS -XPOST -d@layergroup.xml -H "Content-type: text/xml" http://apps.opengeo.org/geoserver/rest/workspaces/osm/layergroups

## GeoServer创建图层

### 创建图层数据表

下载[osmsld.zip][6]

```
wget -O osmsld.zip http://files.cnblogs.com/files/think8848/osmsld.zip

unzip osmsld.zip

su postgres

psql -U think8848 -W -d chinaosmgisdb -a -f /tmp/osmsld/create_tables.sql
```

### 在GeoServer中创建工作空间和数据源(postgis)

### 创建样式和图表

将之前我们下载的osmsld.zip文件中的sld.zip解压开 unzip sld.zip ，然后稍修改下SLD_create.sh文件，主要是修改GeoServer的REST API相关参数

在本文件的最下面，将最后两行暂不用的命令注释掉

然后进入刚才解压sld.zip形成的sld目录 cd sld ，然后调用以下命令

sudo sh /tmp/osmsld/SLD_create.sh

### 创建图层组

打开osmsld.zip包中的layergroup.xml文件，将ocean这一节给删掉,因为并没有导入海图数据

打开SLD_create.sh，参照刚刚注释掉的2行命令创建图层组

```
curl -v -u admin:geoserver -XPOST -d@layergroup.xml -H "Content-type: text/xml" http://localhost:8080/geoserver/rest/workspaces/china/layergroups
```

## 地图乱码问题

安装一个支持中文的字体，如微软雅黑

```
// 创建字体目录，并且将msyh.ttf和msyhbd.ttf复制到字体目录中
sudo mkdir -p /usr/share/fonts/win

sudo mv msyh.ttf msyhbd.ttf /usr/share/fonts/win

// 建立字体索引信息，更新字体缓存
cd /usr/share/fonts/win

sudo mkfontscale

sudo mkfontdir

fc-cache
```

然后把style里面的字体替换

## 导入OpenStreetMap海图数据，并在GeoServer上发布

### 下载[OpenStreetMap海图数据][7]

因为下载的OpenStreetMap的中国数据是Mercator投影坐标系,SRID为3857,s所以下载第二个

### 将shp文件导入到PostGis中

```
su postgres

shp2pgsql -s 3857 -I -D /tmp/water-polygons-split-3857/water_polygons.shp ocean_all | psql -d chinaosmgisdb -U postgres
```

### 将发布的地图范围所在区域(bounds)海图从完整的海图中切出来

```
psql -U postgres -d chinaosmgisdb -W

CREATE TABLE ocean AS
WITH bounds AS (
  SELECT ST_SetSRID(ST_Extent(way)::geometry,3857) AS geom
  FROM planet_osm_line
  )
SELECT 1 AS id, ST_Intersection(b.geom, o.geom) AS geom
FROM bounds b, ocean_all o
WHERE ST_Intersects(b.geom, o.geom);
```

### 在GeoServer中创建ocean图层
直接在GeoServer建图层即可，唯一要注意的就是在图层样式中选择chinaosm:ocean样式

## [Openlayers][8]
Openlayers比较简单，主要难点在于对API的熟练度。官网提供了大量例子来参考

---------------------------------------

参考[think8848的博客](http://think8848.cnblogs.com)

[1]: http://geoserver.org/download/
[2]: http://localhost:8080/geoserver/web
[3]: http://trac.osgeo.org/postgis/wiki/UsersWikiPostGIS23UbuntuPGSQL96Apt
[4]: http://download.geofabrik.de/
[5]: https://codeload.github.com/gravitystorm/openstreetmap-carto/zip/master
[6]: http://files.cnblogs.com/files/think8848/osmsld.zip
[7]: http://openstreetmapdata.com/data/water-polygons
[8]: http://openlayers.org
