## **REGEST 要用的檔案**


```JAVA=
package com.vpon.wizad.etl.pig;

import java.io.IOException;

import com.vpon.wizad.etl.util.GenerateQuadKey;
import org.apache.pig.EvalFunc;
import org.apache.pig.data.Tuple;
import org.apache.pig.data.TupleFactory;
import com.mamind.geoip.LookupService;
import com.mamind.geoip.Location;
import java.util.List;
import java.util.ArrayList;

public class GenerateLocationInfo extends EvalFunc<Tuple> {
    LookupService lookupService;
    GenerateQuadKey quadkeyGenerator;

    public Tuple exec(Tuple input) throws IOException {
        if (lookupService == null) {
            lookupService = new LookupService("./GeoIPCityap.dat",           
             //lookupservervice是把quardkey轉成定義好的區域位置的API
        LookupService.GEOIP_MEMORY_CACHE | LookupService.GEOIP_CHECK_CACHE);
        }
        if (quadkeyGenerator == null) {
            quadkeyGenerator = new GenerateQuadKey();
        }

        try { //用tuple的方式把裡面的資料把起來
            Tuple output = TupleFactory.getInstance().newTuple();
            String ip = (String) input.get(0);
            if (ip.equals("NULL")) {
                return null;
            }
            Location location = lookupService.getLocation(ip);
            output.append(location.countryName);
            output.append(location.city);
            output.append(location.latitude);
            output.append(location.longitude);
            output.append(quadkeyGenerator.generateQuadKey(location.latitude, location.longitude));
            return output;
        } catch (Exception e) {
            return null;
        }
    }

    public List<String> getCacheFiles() {
        List<String> list = new ArrayList<String>(1);
        list.add("/user/cloudera/audi/maxmind/GeoIPCityap.dat#GeoIPCityap.dat");
        return list;
       //GeoIPCityap.dat是quadkey所對應的區域位置(是先定義好的)
    }
}
```


==以上是REGISTER 的檔案，藉由此檔案可以讓輸入近來quarkey轉換成區域位置==

# **add_location_info.pig**
```PIG=
-- UDF
REGISTER /home/cloudera/workspace/audi_UDF.jar;
DEFINE generate_location_info com.vpon.wizad.etl.pig.GenerateLocationInfo();

set default_parallel 2

-- load data
raw = LOAD '/user/cloudera/audi/data/*' USING PigStorage(',')
//數據讀取並以','隔開

//下面為命名欄位 colum_name : data_type
AS (id:chararray, create_time:chararray, action_time:chararray, log_type:int, ad_id:chararray,
positioning_method:int, location_accuracy:double, lat:double, lon:double, cell_id:chararray,
lac:chararray, mcc:chararray, mnc:chararray, ip:chararray, connection_type:int, imei:chararray,
android_id:chararray, udid:chararray, openudid:chararray, idfa:chararray, mac_address:chararray,
uid:chararray, density:double, screen_height:int, screen_width:int, user_agent:chararray,
app_id:chararray, device_model_id:chararray, carrier_id:chararray, os_id:int, os_version:chararray);
-- add location
location_info_added = FOREACH raw GENERATE *, FLATTEN(generate_location_info(ip));

-- store
STORE location_info_added INTO '/user/cloudera/audi/location_info_added/' USING PigStorage(',');
//存置HDFS上
```
:::success
FlATTEN 用法
把group 後裡面內嵌的的table當成每一元素 用for_loop方試提取程每個欄位
![](https://i.imgur.com/tSXHUoU.png)


:::

