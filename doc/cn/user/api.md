# API说明

[English Version](../../en/user/api_en.md)

## 一、API兼容性

TNN所有对外暴露接口均通过PUBLIC宏显示声明，非暴露接口符号均不可见。

```cpp
#if defined _WIN32 || defined __CYGWIN__
  #ifdef BUILDING_DLL
    #ifdef __GNUC__
      #define PUBLIC __attribute__ ((dllexport))
    #else
      #define PUBLIC __declspec(dllexport)
    #endif
  #else
    #ifdef __GNUC__
      #define PUBLIC __attribute__ ((dllimport))
    #else
      #define PUBLIC __declspec(dllimport)
    #endif
  #endif
  #define LOCAL
#else
  #if __GNUC__ >= 4
    #define PUBLIC __attribute__ ((visibility ("default")))
    #define LOCAL  __attribute__ ((visibility ("hidden")))
  #else
    #define PUBLIC
    #define LOCAL
  #endif
#endif
```

不同版本API 兼容性遵守[语义化版本 2.0.0](https://semver.org/lang/zh-CN/)规则。

## 二、API调用

### 简介
API调用主要对模型解析，网络构建，输入设定，输出获取四个步骤进行简要介绍，详细说明参见API详解部分。

### 步骤1. 模型解析

```cpp
TNN tnn;
TNN_NS::ModelConfig model_config;
//proto文件内容存入proto_buffer
model_config.params.push_back(proto_buffer);
//model文件内容存入model_buffer
model_config.params.push_back(model_buffer);
Status ret = tnn.Init(model_config);
```

TNN模型解析需配置ModelConfig params参数，传入proto和model文件内容，并调用TNN Init接口即可完成模型解析。

### 步骤2. 网络构建

```cpp
TNN_NS::NetworkConfig config;
config.device_type = TNN_NS::DEVICE_ARM;
TNN_NS::Status error;
auto net_instance = tnn.CreateInst(config, error);
```

TNN网络构建需配置NetworkConfig，device_type可配置`DEVICE_ARM`， `DEVICE_OPENCL`， `DEVICE_METAL`， `DEVICE_X86`， `DEVICE_CUDA`, `DEVICE_HUAWEI_NPU`, `DEVICE_RK_NPU`等多种加速方式，通过CreateInst接口完成网络的构建。


### 步骤3. 输入设定

```cpp
auto status = instance->SetInputMat(input_mat, input_cvt_param);
```

TNN输入设定通过调用SetInputMat接口完成，需要传入的数据保存在`input_mat`中。

### 步骤4. 网络运行

```cpp
auto status = instante->Forward();
```
TNN Forward接口为同步调用接口，ForwardAsync接口为异步调用接口。

### 步骤5. 输出获取

```cpp
auto status = instance->GetOutputMat(output_mat);
```

TNN输出获取通过调用GetOutputMat接口完成，输出结果将按照特定格式保存在`output_mat`中。

## 二、API详解

### API目录结构

```bash
.
└── tnn
    ├── core
    │   ├── blob.h                  # 负责数据传递
    │   ├── common.h                # 定义常用结构
    │   ├── instance.h              # 网络实例
    │   ├── macro.h                 # 常用宏定义
    │   ├── mat.h                   # 输入接口，类cv::Mat
    │   ├── status.h                # 接口状态
    │   └── tnn.h                   # 模型解析
    ├── utils
    │   ├── bfp16_utils.h           # bfp16转换工具
    │   ├── blob_converter.h        # blob输入输出转换工具
    │   ├── cpu_utils.h             # CPU性能特定优化工具
    │   ├── data_type_utils.h       # 数据类型转换工具
    │   ├── dims_vector_utils.h     # 尺寸计算工具
    │   ├── half_utils.h            # fp16转换工具
    │   ├── mat_utils.h             # Mat转换工具
    │   └── string_utils.h          # 字符串转换工具
    └── version.h                   # 编译构建信息
```

### 1. core/common.h

`DataType`：定义不同数据类型枚举值。  
`DataFormat`：定义Blob Data不同数据排布方式。  
`NetworkType`：定义不同网络构建类型，默认构建TNN网络，支持第三方库网络构建。  
`DeviceType`：用于指定网络运行设备及加速方式。  
`ModelType`：定义模型类型，TNN默认解析模型为TNN模型，同时支持其他第三方库模型格式传入。  
`Precision `: 定义网络运行精度。    

```cpp
struct PUBLIC ModelConfig {

    ModelType model_type = MODEL_TYPE_TNN;

    // tnn model need two params: order is proto content, model content.
    // ncnn need two: params: order is param content, bin content.
    // openvino model need two params: order is xml content, model path.
    // coreml model need one param: coreml model directory path.
    // snpe model need one param: dlc model directory path.
    // hiai model need two params: order is model name, model file path.
    // atlas model need one param: config string.
    std::vector<std::string> params;
};
```

ModelConfig参数说明：

- `model_type`: TNN当前开源版本仅支持传入`MODEL_TYPE_TNN`， `MODEL_TYPE_NCNN`, `MODEL_TYPE_COREML` 模型格式。  
- `params`: TNN模型需传入proto文件内容以及model文件路径。NCNN模型需传入param文件内容以及bin文件路径, COREML模型需传入coreml 模型所在目录路径。


```cpp
struct PUBLIC NetworkConfig {
    // device type default cpu
    DeviceType device_type = DEVICE_ARM;

    // device id default 0
    int device_id = 0;

    // blob data format, auto decided by device
    DataFormat data_format = DATA_FORMAT_AUTO;

    // network type, auto decided by device
    NetworkType network_type = NETWORK_TYPE_AUTO;

    // raidnet instances not share memory with others
    ShareMemoryMode share_memory_mode = SHARE_MEMORY_MODE_DEFAULT;

    // dependent library path
    std::vector<std::string> library_path = {};

    // compute precision
    Precision precision = PRECISION_AUTO;

    // cache path to store possible cache models or opt kernel or opencl program cache
    std::string cache_path = "";

    // network init or reshape may cost more time to select opt kernel implement if enable tune kernel
    // cache_path can set to store tune kernel info.
    bool enable_tune_kernel = false;
};
```

NetworkConfig参数说明：  

- `device_type`: 默认为`DEVICE_ARM`。 当前已支持 `DEVICE_NAIVE`、`DEVICE_ARM`、`DEVICE_X86`、`DEVICE_OPENCL`、`DEVICE_METAL`、`DEVICE_CUDA`、`DEVICE_HUAWEI_NPU`、`DEVICE_RK_NPU`。  
- `device_id`: 默认为0，多个设备支持通过`device_id`选择，当前仅`DEVICE_CUDA`需配置此参数指定gpu id。  
- `data_format`: 默认为tnn自动选择blob数据排布方式进行加速，可通过此参数设定特定blob数据排布进行加速。  
- `network_type`: 默认根据`device_type`自动选择网络类型，可指定构建网络类型。  
- `share_memory_mode`: tnn instance 内存共享方式。  
- `library_path`: 支持外部依赖库加载，iOS metal kernel库放在app非默认路径需配置此参数。    
- `precision`:  网络精度类型，默认根据不同的`device_type`自动选择精度。  
- `cache_path`： 华为NPU指定cache路径可存放运行过程中转出的om文件，后续运行可直接通过加载cache路径对应om文件。OpenCL指定cache路径可缓存编译好的kernel二进制文件，后续初始化可直接通过二进制cache文件创建kernel， `enable_tune_kernel` 打开，可通过指定cache路径存放tune参数，后续可直接加载tune参数而无需每次运行都tune kernel。


```cpp
typedef enum {
    // default
    SHARE_MEMORY_MODE_DEFAULT = 0,
    // same thread tnn instance share blob memory
    SHARE_MEMORY_MODE_SHARE_ONE_THREAD = 1,
    // set blob memory from external, different thread share blob memory need
    // synchronize
    SHARE_MEMORY_MODE_SET_FROM_EXTERNAL = 2
} ShareMemoryMode;
```

ShareMemoryMode参数说明:  

- `SHARED_MEMORY_MODE_DEFAULT`: 仅支持同一instance不同blob间内存共享。  
- `SHARE_MEMORY_MODE_SHARE_ONE_THREAD`: 支持同一线程的不同Instance内存共享。  
- `SHARE_MEMORY_MODE_SET_FROM_EXTERNAL`: 支持instance内存由外部传入，共享方式由调用侧决定，线程间共享需处理同步问题，内存分配释放均需调用侧维护。  

### 2. core/tnn.h

```cpp
class PUBLIC TNN {
public:
    ...

    // init tnn implement, interpret model.
    Status Init(ModelConfig& config);

    // denit tnn implement, release model interpreter.
    Status DeInit();

    // add output to the model.
    // if output_name of blob not found, then search output_index of layer.
    Status AddOutput(const std::string& output_name, int output_index = 0);

    // return input shapes map from model
    Status GetModelInputShapesMap(InputShapesMap& shapes_map);

    // create tnn network instance with network config and inputs shape.
    // if inputs shape not set, use default from model.
    std::shared_ptr<Instance> CreateInst(
        NetworkConfig& config, Status& status,
        InputShapesMap inputs_shape = InputShapesMap());

    // create tnn network instance with network config and min max inputs shape,
    // instance reshape can support range from min inputs shape to max inputs shape.
    std::shared_ptr<Instance> CreateInst(
        NetworkConfig& config, Status& status,
        InputShapesMap min_inputs_shape, InputShapesMap max_inputs_shape);

    ...
};
```

TNN接口说明：  

- Init接口：负责模型数据传入并解析，需配置并传入ModelConfig。  
- DeInit接口: 负责tnn implement释放，默认析构函数可自动释放。  
- AddOutput接口：支持增加模型输出，可将网络任意一层输出定义为模型输出。
- GetModelInputShapesMap接口： 获取模型解析出的模型输入尺寸。  
- CreateInst接口：负责网络实例Instance构建，如果运行过程中支持输入维度可变，需配置`min_inputs_shape`和`max_inputs_shape`指定输入每个维度支持的最大最小尺寸。

### 3. core/instance.h

```cpp
class PUBLIC Instance {
public:
    Instance(NetworkConfig& net_config, ModelConfig& model_config);

    ~Instance();

    // init with model interpeter and inputs shape.
    Status Init(std::shared_ptr<AbstractModelInterpreter> interpreter, InputShapesMap inputs_shape);

    // deinit, release network
    Status DeInit();

    //  return memory bytes required for forward
    Status GetForwardMemorySize(int& memory_size);

    //  set memory to tnn instance. if success, return status code zero.
    //  only instance created with SHARE_MEMORY_MODE_SET_FROM_EXTERNAL can be set from external.
    //  the memory size need >=  GetForwardMemorySize().
    //  releasing or otherwise using the memory for other purposes during the tnn network run 
    //  will result in undefined behavior.
    Status SetForwardMemory(void* memory);

    // reshape instance with new input shapes
    Status Reshape(const InputShapesMap& inputs);

    // get tnn command queue
    Status GetCommandQueue(void** command_queue);

    // @brief tnn instance network infer, it will wait until all layer infer complete.
    Status Forward();

    ...

    // tnn instance network infer async.
    // device gpu, all layer infer complete will call Callback.
    Status ForwardAsync(Callback call_back);

    // get all input blobs
    Status GetAllInputBlobs(BlobMap& blobs);

    // get all output blobs
    Status GetAllOutputBlobs(BlobMap& blobs);

    // set threads run on cpu 
    virtual Status SetCpuNumThreads(int num_threads);
    ...

    // set input Mat, if input_name is not set, take the first input as default
    Status SetInputMat(std::shared_ptr<Mat> mat,
                       MatConvertParam param,
                       std::string input_name = "");
    
    // get output Mat, if output_name is not set, take the first output as default
    Status GetOutputMat(std::shared_ptr<Mat>& mat,
                        MatConvertParam param = MatConvertParam(),
                        std::string output_name = "", 
                        DeviceType device = DEVICE_ARM, MatType mat_type = NCHW_FLOAT);

};
```

Instance接口说明：  

- `Instance`和`Init`接口均由TNN CreateInst接口实现调用，用于生成Instance网络实例。  
- `GetForwardMemorySize`可获取Instance所有Blob所需内存大小，`SetForwardMemory`用于传入外部内存。对于`SHARE_MEMORY_MODE_SET_FROM_EXTERNAL`内存模式构建的Instance，内存需由外部传入， 传入内存实际大小不得小于`GetForwardMemorySize`返回值大小。  
- `Reshape`接口支持网络构建成功后重新设定输入尺寸，仅通过`min_inputs_shape`和`max_inputs_shape` 构建的网络可在运行过程中改变输入尺寸，可变尺寸范围由`min_inputs_shape`和`max_inputs_shape` 指定。  
- `GetCommandQueue`接口支持获取网络运行对应的command queue，同一command queue消息顺序执行。  
- `GetAllInputBlobs`和 `GetAllOutputBlobs`分别用于获取输入输出blob。  
- `SetCpuNumThreads`可设置CPU线程并行数。  
- `Forward`为网络运行同步接口，`ForwardAsync`为网络运行异步接口。  
- `SetInputMat`用于设定输入Mat，其中MatConvertParam可设定[转换参数](#MatConvertParam参数说明)。对于多输入网络，可用`input_name`区分。  
- `GetOutputMat`用于获取输出结果并保存在输出Mat中，其中MatConvertParam可设定[转换参数](#MatConvertParam参数说明)。对于多输出网络，可用`output_name`区分，DeviceType可指定输出Mat Memory构建在CPU还是GPU，MatType可用于设定输出Mat数据排列方式。  


### 4. core/mat.h

```cpp
class PUBLIC Mat {
public:
    ...

    Mat(DeviceType device_type, MatType mat_type, DimsVector shape_dims, void* data);
    Mat(DeviceType device_type, MatType mat_type, DimsVector shape_dims);
    //empty mat
    Mat(DeviceType device_type, MatType mat_type);

    DEPRECATED("use Mat(DeviceType, MatType, DimsVector, void*) instead")
    Mat(DeviceType device_type, MatType mat_type, void* data) : Mat(device_type, mat_type, {1,0,0,0}, data) {};
    
    ...
};
```
其中MatType支持常用的CV, NLP输入输出布局，且`DeviceType`可设定为CPU，GPU。

```cpp
typedef enum {
    INVALID    = -1,
    //bgr or rgb: uint8
    N8UC3      = 0x00,
    //bgra or rgba: uint8
    N8UC4      = 0x01,
    //gray: uint8
    NGRAY      = 0x10,
    //YUV420SP, YYYYVUVUVU
    NNV21      = 0x11,
    //YUV420SP, YYYYUVUVUV
    NNV12      = 0x12,
    //nchw: float
    NCHW_FLOAT = 0x20,
    // nchw: int32
    NC_INT32 = 0x21,
    //ncdhw: float
    NCDHW_FLOAT = 0x100,
    ...
} PUBLIC MatType;
```

### 5. core/macro.h
提供不同平台Log宏，不同数据类型最大最小值宏，PUBLIC宏定义，以及部分数据pack转换等宏定义。

### 6. core/status.h
`Status`定义于status.h头文件中。

```cpp
enum StatusCode {

    TNN_OK = 0x0,

    // param errcode
    TNNERR_PARAM_ERR        = 0x1000,
    TNNERR_INVALID_NETCFG   = 0x1002,
    ...
}

class PUBLIC Status {
public:
    Status(int code = TNN_OK, std::string message = "OK");

    Status &operator=(int code);

    bool operator==(int code_);
    bool operator!=(int code_);
    operator int();
    operator bool();
    std::string description();

private:
    int code_;
    std::string message_;
}
```
当Status code不为TNN_OK，通过`description`接口可返回错误描述信息。

### 7. core/blob.h

```cpp
// @brief BlobDesc blob data info
struct PUBLIC BlobDesc {
    // deivce_type describes devie cpu, gpu, ...
    DeviceType device_type = DEVICE_NAIVE;
    // data_type describes data precion fp32, in8, ...
    DataType data_type = DATA_TYPE_FLOAT;
    // data_format describes data order nchw, nhwc, ...
    DataFormat data_format = DATA_FORMAT_AUTO;
    // DimsVector describes data dims
    DimsVector dims;
    // name describes the blob name
    std::string name;
    
    std::string description(bool all_messgae = false);
};

struct PUBLIC BlobHandle {
    void *base            = NULL;
    uint64_t bytes_offset = 0;
};

// @brief Blob tnn data store and transfer interface.
class PUBLIC Blob {
public:
    ...

    //@brief create Blob with blob descript and data handle
    Blob(BlobDesc desc, BlobHandle handle);

    ...
};

```

Blob当前主要由`BlobDesc`以及`BlobHandle`构成，其中`BlobDesc`描述Blob相关结构信息，`BlobHandle`用于读取和存储Blob数据。

`BlobDesc`用于描述`device_type`, `data_type`, `data_format`, `dims`, `name`信息。

dims描述blob维度信息，dims存储尺寸与data_format无关：  
- dims尺寸为2，存储对应N, C。  
- dims尺寸为4，存储尺寸对应N，C，H，W。  
- dims尺寸为5，存储尺寸对应N，C，D，H，W。  

当前不同平台blob输入输出数据类型及排布如下：  

- `ARM`：CPU内存， NC4HW4.  
- `OPENCL`: GPU显存（clImage）， NHC4W4. 其中NH为clImage高，C4W4为clImage宽。  
- `METAL`: GPU显存（metal)， NC4HW4.  
- `HUAWEI_NPU`: CPU内存, NCHW.
- `X86`： CPU内存，NCHW。  
- `CUDA`： GPU内存， NCHW。  

其中最后4代表pack 4, C4代表最后1位4由4个C进行pack。  


### 8. utils/mat\_utils.h
```cpp
class PUBLIC MatUtils {
public:
    //copy cpu <-> device, cpu<->cpu, device<->device, src and dst dims must be equal.
    static Status Copy(Mat& src, Mat& dst, void* command_queue);

    //src and dst device type must be same. when param scale_w or scale_h is 0, it is computed as
    // (double)dst.GetWidth() / src.GetWidth() or (double)dst.GetHeight() / src.GetHeight().
    static Status Resize(Mat& src, Mat& dst, ResizeParam param, void* command_queue);

    //src and dst device type must be same. when param width or height is 0, it is equal to
    //dst.GetWidth() or dst.GetHeight().
    static Status Crop(Mat& src, Mat& dst, CropParam param, void* command_queue);

    //src and dst device type must be same.
    static Status WarpAffine(Mat& src, Mat& dst, WarpAffineParam param, void* command_queue);

    //src and dst device type must be same.
    static Status CvtColor(Mat& src, Mat& dst, ColorConversionType type, void* command_queue);

    //src and dst device type must be same. param top, bottom, left and right must be non-negative.
    static Status CopyMakeBorder(Mat& src, Mat& dst, CopyMakeBorderParam param, void* command_queue);
};
```

接口参数说明:

- `Copy`: 支持不同DEVICE与CPU Mat数据拷贝，以及相同DEVICE间Mat数据拷贝。
- `Resize `、`Crop`、`WarpAffine `、`CvtColor `、`CopyMakeBorder` 接口行为类似OpenCV，CPU与GPU均支持，`src` 和  `dst` 需拥有相同的`DEVICE_TYPE`。


### 9. utils/bfp16\_utils.h
接口提供了cpu内存fp32和bfp16转换工具。  

### 10. utils/blob\_convert.h
```cpp
class PUBLIC BlobConverter {
public:
    explicit BlobConverter(Blob* blob);
    virtual Status ConvertToMat(Mat& image, MatConvertParam param, void* command_queue);
    virtual Status ConvertFromMat(Mat& image, MatConvertParam param, void* command_queue);

    virtual Status ConvertToMatAsync(Mat& image, MatConvertParam param, void* command_queue);
    virtual Status ConvertFromMatAsync(Mat& image, MatConvertParam param, void* command_queue);

private:
    Blob* blob_;
    std::shared_ptr<BlobConverterAcc> impl_ = nullptr;
};
```

通过`ConvertToMat`可将blob数据按照Mat格式传入Mat，`ConvertFromMat`可将Mat数据按照blob格式传入blob, 接口对应的`command_queue`可通过 Instance `GetCommandQueue`接口获取。

接口提供常用预处理，后处理支持，支持设定scale, bias参数以及reverse channel适配bgr, rgb等场景。

```cpp
struct PUBLIC MatConvertParam {
    std::vector<float> scale = {1.0f, 1.0f, 1.0f, 1.0f};
    std::vector<float> bias = {0.0f, 0.0f, 0.0f, 0.0f};
    bool reverse_channel = false;
};
```

#### MatConvertParam参数说明：  
- `reverse_channel`: 默认为`false`，若需要交换图像的B和R维度，可将此参数设置为`true`。  
    * 仅`N8UC3`和`N8UC4`类型的Mat支持reverse_channel，其他类型的Mat会忽略该参数。  
    * `ConvertFromMat`和`ConvertToMat`过程都支持reverse_channel。  
- `scale`和`bias`: scale默认为 `1`，bias默认为`0`，计算顺序为先乘scale，再加bias。  
    * 所有类型的Mat都支持scale和bias。  
    * `ConvertFromMat`和`ConvertToMat`过程都支持scale和bias。  
    * 若指定的scale全为`1`，且bias全为`0`，或者使用默认的scale和bias值，则不做乘scale和加bias操作；否则用户需提供与channel大小对应的scale和bias值。  
    * 对于多维数据，scale和bias中的数值顺序和推理过程使用的数据格式保持一致。例如，若模型实际使用BGR格式进行推理，则`ConvertFromMat`和`ConvertToMat`过程，无论reverse_channel与否，scale和bias都需按照BGR顺序指定。也可理解为，`ConvertFromMat`先reverse channel，再乘scale和加bias；`ConvertToMat`先乘scale和加bias，再reverse channel。  

### 11. utils/cpu\_utils.h
提供CPU线程核绑定以及省电模式等设定相关工具。

### 12. utils/data\_type\_utils.h
提供DataType尺寸和名称转换相关工具。

### 13. utils/dims\_vector\_utils.h
提供常用blob dims计算比较工具。

### 14. utils/half\_utils.h
接口提供cpu内存fp32和fp16转换工具。

### 15. utils/string\_utils.h
接口提供uchar string 到std::string的转换，主要用于TNN模型内存输入。

### 16. version.h
构建版本信息
