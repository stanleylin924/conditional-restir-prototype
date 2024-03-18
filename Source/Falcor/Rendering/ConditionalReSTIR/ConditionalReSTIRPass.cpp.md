```cpp
#include "ConditionalReSTIRPass.h"
#include "Core/Assert.h"
#include "Core/API/RenderContext.h"
#include "Utils/Logger.h"
#include "Utils/Timing/Profiler.h"
#include "Utils/Color/ColorHelpers.slang"
#include "../../../RenderPasses/PathTracer/Params.slang"
```
```cpp
namespace Falcor
{
    namespace
    {
        const char kReflectTypesFile[] = "Rendering/ConditionalReSTIR/ReflectTypes.cs.slang";
        const char kSuffixSpatialResamplingFile[] = "Rendering/ConditionalReSTIR/SuffixResampling.cs.slang";
        const char kSuffixTemporalResamplingFile[] = "Rendering/ConditionalReSTIR/SuffixResampling.cs.slang";
        const char kSuffixResamplingFile[] = "Rendering/ConditionalReSTIR/SuffixResampling.cs.slang";
        const char kTemporalResamplingFile[] = "Rendering/ConditionalReSTIR/TemporalResampling.cs.slang";
        const char kSpatialResamplingFile[] = "Rendering/ConditionalReSTIR/SpatialResampling.cs.slang";
        const char kTemporalRetraceFile[] = "Rendering/ConditionalReSTIR/TemporalPathRetrace.cs.slang";
        const char kSpatialRetraceFile[] = "Rendering/ConditionalReSTIR/SpatialPathRetrace.cs.slang";
        const char kProduceRetraceWorkload[] = "Rendering/ConditionalReSTIR/ProduceRetraceWorkload.cs.slang";
        const char kSuffixRetraceFile[] = "Rendering/ConditionalReSTIR/SuffixPathRetrace.cs.slang";
        const char kSuffixProduceRetraceWorkload[] = "Rendering/ConditionalReSTIR/SuffixProduceRetraceWorkload.cs.slang";
        const char kSuffixRetraceTalbotFile[] = "Rendering/ConditionalReSTIR/SuffixPathRetraceTalbot.cs.slang";
        const char kSuffixProduceRetraceTalbotWorkload[] = "Rendering/ConditionalReSTIR/SuffixProduceRetraceTalbotWorkload.cs.slang";
```
- 這些是用於條件性 ReSTIR（ConditionalReSTIR）渲染器的文件路徑。這些路徑都是字符串常量，指向不同的 Slang 文件，用於實現條件性 ReSTIR 算法的不同部分。以下是這些文件的簡要說明：

    - kReflectTypesFile：定義了用於條件性 ReSTIR 的反射類型。
    - kSuffixSpatialResamplingFile：空間重抽樣算法的 Slang 文件。
    - kSuffixTemporalResamplingFile：時間重抽樣算法的 Slang 文件。
    - kSuffixResamplingFile：重抽樣算法的通用 Slang 文件。
    - kTemporalResamplingFile：時間重抽樣算法的 Slang 文件。
    - kSpatialResamplingFile：空間重抽樣算法的 Slang 文件。
    - kTemporalRetraceFile：時間路徑重追蹤算法的 Slang 文件。
    - kSpatialRetraceFile：空間路徑重追蹤算法的 Slang 文件。
    - kProduceRetraceWorkload：生成路徑重追蹤工作負載的 Slang 文件。
    - kSuffixRetraceFile：路徑重追蹤算法的通用 Slang 文件。
    - kSuffixProduceRetraceWorkload：生成路徑重追蹤工作負載的通用 Slang 文件。
    - kSuffixRetraceTalbotFile：Talbot 路徑重追蹤算法的通用 Slang 文件。
    - kSuffixProduceRetraceTalbotWorkload：生成 Talbot 路徑重追蹤工作負載的通用 Slang 文件。

- 這些文件構成了條件性 ReSTIR 渲染器的不同組件，用於實現其核心算法和功能。

```cpp
        const char kPrefixRetraceFile[] = "Rendering/ConditionalReSTIR/PrefixPathRetrace.cs.slang";
        const char kPrefixProduceRetraceWorkload[] = "Rendering/ConditionalReSTIR/PrefixProduceRetraceWorkload.cs.slang";
        const char kPrefixResampling[] = "Rendering/ConditionalReSTIR/PrefixResampling.cs.slang";
```
- 這些是用於條件性 ReSTIR（ConditionalReSTIR）渲染器的文件路徑和一個字符串常量。以下是它們的簡要說明：

    - kPrefixRetraceFile：路徑重追蹤算法的前綴（Prefix）部分的 Slang 文件路徑。
    - kPrefixProduceRetraceWorkload：生成路徑重追蹤工作負載的前綴部分的 Slang 文件路徑。
    - kPrefixResampling：前綴重抽樣算法的 Slang 文件路徑。

- 這些文件包含了條件性 ReSTIR 渲染器的不同組件，用於實現其核心算法和功能。

```cpp
        const char kTraceNewSuffixes[] = "Rendering/ConditionalReSTIR/TraceNewSuffixes.cs.slang";
        const char kPrefixNeighborSearch[] = "Rendering/ConditionalReSTIR/PrefixNeighborSearch.cs.slang";
        const char kTraceNewPrefixes[] = "Rendering/ConditionalReSTIR/TraceNewPrefixes.cs.slang";
```
- 這些是與條件性 ReSTIR 渲染器相關的特定功能的 Slang 文件路徑：

    - kTraceNewSuffixes：用於追蹤新的後綴路徑的 Slang 文件路徑。
    - kPrefixNeighborSearch：用於前綴鄰域搜索的 Slang 文件路徑。
    - kTraceNewPrefixes：用於追蹤新的前綴路徑的 Slang 文件路徑。

- 這些文件包含了條件性 ReSTIR 渲染器中特定部分的實現，例如路徑追蹤和搜索算法。

```cpp
        const std::string kShaderModel = "6_5";
```
- kShaderModel 是一個字符串常量，表示所使用的 HLSL（High-Level Shading Language）或者 Slang 的版本，此處是版本為 6.5。

```cpp
        const Gui::DropdownList kShiftMappingList =
        {
            { (uint32_t)ConditionalReSTIR::ShiftMapping::Reconnection, "Reconnection" },
            { (uint32_t)ConditionalReSTIR::ShiftMapping::Hybrid, "Hybrid" },
        };

        const Gui::DropdownList kRetraceScheduleType =
        {
            { (uint32_t)ConditionalReSTIR::RetraceScheduleType::Naive, "Naive" },
            { (uint32_t)ConditionalReSTIR::RetraceScheduleType::Compact, "Compact" },
        };

        const Gui::DropdownList kKNNAdaptiveRadiusType = {
            {(uint32_t)ConditionalReSTIR::KNNAdaptiveRadiusType::NonAdaptive, "NonAdaptive"},
            {(uint32_t)ConditionalReSTIR::KNNAdaptiveRadiusType::RayCone, "RayCone"},
        };
```
- 這些是用於 GUI 下拉列表的選項列表，具體說明如下：

    - kShiftMappingList：包含了條件性 ReSTIR 渲染器中移位映射方法的選項列表。這些選項包括“Reconnection”和“Hybrid”。
    - kRetraceScheduleType：包含了條件性 ReSTIR 渲染器中追蹤排程類型的選項列表。這些選項包括“Naive”和“Compact”。
    - kKNNAdaptiveRadiusType：包含了條件性 ReSTIR 渲染器中 KNN 自適應半徑類型的選項列表。這些選項包括“NonAdaptive”和“RayCone”。

```cpp
        const uint32_t kNeighborOffsetCount = 8192;
    }
```
- 這是一個常量，表示鄰接點偏移量的數量，其值為 8192。在某些上下文中，這可能用於定義鄰接點的最大數量或偏移量的緩衝區大小。

```cpp
    ConditionalReSTIRPass::SharedPtr ConditionalReSTIRPass::create(const Scene::SharedPtr& pScene, const Program::DefineList& ownerDefines, const Options& options, const PixelStats::SharedPtr& pPixelStats)
    {
        return SharedPtr(new ConditionalReSTIRPass(pScene, ownerDefines, options, pPixelStats));
    }
```
- 這段程式碼是一個靜態成員函式的定義，用於創建條件性 ReSTIR (Reconstruction and Shading with Integrated Ray tracing in Real-time) 通道的實例。該函式通常位於 ConditionalReSTIRPass 類別中，用於動態地建立這個通道的實例。該函式接受多個參數，包括場景指針 (pScene)、宿主定義列表 (ownerDefines)、選項 (options) 和像素統計 (pPixelStats)，並返回一個共享指標指向創建的 ConditionalReSTIRPass 物件。

```cpp
    void ConditionalReSTIRPass::createComputePass(ComputePass::SharedPtr& pPass, std::string shaderFile, Program::DefineList defines, Program::Desc baseDesc, std::string entryFunction)
    {
        if (!pPass)
        {
            Program::Desc desc = baseDesc;
            desc.addShaderLibrary(shaderFile).csEntry(entryFunction == "" ? "main" : entryFunction);
            pPass = ComputePass::create(desc, defines, false);
        }
        pPass->getProgram()->addDefines(defines);
        pPass->setVars(nullptr);
    }
```
- 這是一個靜態成員函式 createComputePass 的定義，屬於 ConditionalReSTIRPass 類別。這個函式用於創建計算通道（Compute Pass）。它接受以下參數：

    - pPass：計算通道的指針。如果為空，則會創建一個新的計算通道。
    - shaderFile：包含計算着色器的文件路徑。
    - defines：定義列表，用於定義計算着色器。
    - baseDesc：計算通道的基本描述。
    - entryFunction：計算着色器的入口函式名稱。

- 此函式首先檢查 pPass 是否為空。如果是空的，則創建一個新的計算通道。然後，將指定的着色器文件添加到計算通道的描述中，並設置入口函式。接著，將定義添加到計算通道的程序中，最後將計算通道的變數設置為空。

```cpp
    ConditionalReSTIRPass::ConditionalReSTIRPass(const Scene::SharedPtr& pScene, const Program::DefineList& ownerDefines, const Options& options, const PixelStats::SharedPtr& pPixelStats)
        : mpScene(pScene)
        , mOptions(options)
    {
        FALCOR_ASSERT(mpScene);

        mpPixelDebug = PixelDebug::create();

        // Create compute pass for reflecting data types.
        Program::Desc desc;
        Program::DefineList defines;
        defines.add(mpScene->getSceneDefines());
        defines.add(ownerDefines);
        desc.addShaderLibrary(kReflectTypesFile).csEntry("main").setShaderModel(kShaderModel);
        mpReflectTypes = ComputePass::create(desc, defines);

        // Create neighbor offset texture.
        mpNeighborOffsets = createNeighborOffsetTexture(kNeighborOffsetCount);

        mpPixelStats = pPixelStats;
    }
```
- 這是一個名為 ConditionalReSTIRPass 的類別的構造函數。這個類別負責實現一個特定的渲染通道或過程，稱為 "Conditional ReSTIR"。在構造函數中，以下是執行的一些主要操作：

    1. 初始化成員變量：初始化了指向場景（pScene）、像素統計（pPixelStats）的指針，以及渲染選項（options）。
    2. 創建像素調試（PixelDebug）：創建了用於像素調試的對象。該對象可能用於在渲染過程中檢查和調試像素級別的數據。
    3. 創建反射數據類型的計算通道：使用指定的着色器文件（kReflectTypesFile）和着色器入口點（main）創建了一個計算通道，該通道負責生成反射數據類型。
    4. 創建鄰接偏移紋理：創建了一個用於存儲鄰接偏移的紋理，這個紋理是根據指定的偏移計算數量創建的（kNeighborOffsetCount）。

- 總之，這個構造函數初始化了一個名為 ConditionalReSTIRPass 的渲染通道，並準備了相關的計算資源，以便在後續的渲染過程中使用。

```cpp
    Program::DefineList ConditionalReSTIRPass::getDefines() const
    {
        Program::DefineList defines;
        defines.add("TEMPORAL_UPDATE_FOR_DYNAMIC_SCENE", mOptions.temporalUpdateForDynamicScene ? "1": "0");
        defines.add("USE_RESERVOIR_COMPRESSION", mOptions.useReservoirCompression ? "1" : "0");
        defines.add("RETRACE_SCHEDULE_TYPE", std::to_string((uint32_t)mOptions.retraceScheduleType));
        defines.add("COMPRESS_PREFIX_SEARCH_ENTRY", mOptions.subpathSetting.compressNeighborSearchKey ? "1" : "0");
        defines.add("USE_PREV_FRAME_SCENE_DATA", mOptions.usePrevFrameSceneData ? "1" : "0");

        return defines;
    }
```
 這個函數名為 getDefines()，它返回一個 Program::DefineList 對象，其中包含了一系列用於定義預處理器符號的指令。這些預處理器符號用於在後續的編譯過程中控制編譯器的行為，以及定義一些編譯時的參數和選項。

- 具體來說，這個函數執行以下操作：

    - 如果 mOptions.temporalUpdateForDynamicScene 為真，則將 "TEMPORAL_UPDATE_FOR_DYNAMIC_SCENE" 定義為 1，否則定義為 0。這個符號可能用於控制動態場景的時域更新。
    - 如果 mOptions.useReservoirCompression 為真，則將 "USE_RESERVOIR_COMPRESSION" 定義為 1，否則定義為 0。這個符號可能用於控制是否使用儲氣庫壓縮。
    - 將 "RETRACE_SCHEDULE_TYPE" 定義為 mOptions.retraceScheduleType 的整數值。這個符號可能用於定義重追蹤的計劃類型。
    - 如果 mOptions.subpathSetting.compressNeighborSearchKey 為真，則將 "COMPRESS_PREFIX_SEARCH_ENTRY" 定義為 1，否則定義為 0。這個符號可能用於控制是否對前綴搜索鍵進行壓縮。
    - 如果 mOptions.usePrevFrameSceneData 為真，則將 "USE_PREV_FRAME_SCENE_DATA" 定義為 1，否則定義為 0。這個符號可能用於控制是否使用前一幀的場景數據。

- 總之，這個函數返回了一系列用於控制編譯時行為的預處理器符號，這些符號將影響後續的計算過程中使用的程序和資源。

---
- 這段程式碼是一個函式 ConditionalReSTIRPass::setShaderData，它用於設置著色器數據。它通過讀取 mOptions 中的值並將其設置為給定 ShaderVar 變量中的相應字段。這些字段包括各種設置和參數，用於控制光線追蹤和預渲染方面的行為，如本地策略類型、鏡面粗糙度閾值、近場距離閾值等。其中一些值是浮點數，一些是整數，還有一些是布林值。最後，它將一些數據，如場景半徑、每像素樣本數等，設置為給定的變量。
```cpp
    void ConditionalReSTIRPass::setShaderData(const ShaderVar& var) const
    {
        var["settings"]["localStrategyType"] = mOptions.shiftMappingSettings.localStrategyType;
        var["settings"]["specularRoughnessThreshold"] = mOptions.shiftMappingSettings.specularRoughnessThreshold;
        var["settings"]["nearFieldDistanceThreshold"] = mOptions.shiftMappingSettings.nearFieldDistanceThreshold;
```
- 這部分代碼將 C++ 程序中的 mOptions 中的 shiftMappingSettings 中的特定設置值傳遞給 shader 中的 settings 變量，這些值包括：

    - localStrategyType：指定了局部策略的類型，可能是某種移位映射算法的一個參數。
    - specularRoughnessThreshold：指定了對於鏡面粗糙度的閾值，可能是移位映射中的一個參數。
    - nearFieldDistanceThreshold：指定了近場距離的閾值，可能是移位映射中的一個參數。

```cpp
        var["subpathSettings"]["adaptivePrefixLength"] = mOptions.subpathSetting.adaptivePrefixLength;
        var["subpathSettings"]["avoidSpecularPrefixEndVertex"] = mOptions.subpathSetting.avoidSpecularPrefixEndVertex;
        var["subpathSettings"]["avoidShortPrefixEndSegment"] = mOptions.subpathSetting.avoidShortPrefixEndSegment;
        var["subpathSettings"]["shortSegmentThreshold"] = mOptions.subpathSetting.shortSegmentThreshold;

        var["subpathSettings"]["suffixSpatialNeighborCount"] = mOptions.subpathSetting.suffixSpatialNeighborCount;
        var["subpathSettings"]["suffixSpatialReuseRadius"] = mOptions.subpathSetting.suffixSpatialReuseRadius;
        var["subpathSettings"]["suffixSpatialReuseRounds"] = mOptions.subpathSetting.suffixSpatialReuseRounds;
        var["subpathSettings"]["numIntegrationPrefixes"] = mOptions.subpathSetting.numIntegrationPrefixes;
        var["subpathSettings"]["generateCanonicalSuffixForEachPrefix"] = mOptions.subpathSetting.generateCanonicalSuffixForEachPrefix;

        var["subpathSettings"]["suffixTemporalReuse"] = mOptions.subpathSetting.suffixTemporalReuse;
        var["subpathSettings"]["temporalHistoryLength"] = mOptions.subpathSetting.temporalHistoryLength;

        var["subpathSettings"]["prefixNeighborSearchRadius"] = mOptions.subpathSetting.prefixNeighborSearchRadius;
        var["subpathSettings"]["prefixNeighborSearchNeighborCount"] = mOptions.subpathSetting.prefixNeighborSearchNeighborCount;
        var["subpathSettings"]["finalGatherSuffixCount"] = mOptions.subpathSetting.finalGatherSuffixCount;

        var["subpathSettings"]["useTalbotMISForGather"] = mOptions.subpathSetting.useTalbotMISForGather;
        var["subpathSettings"]["nonCanonicalWeightMultiplier"] = mOptions.subpathSetting.nonCanonicalWeightMultiplier;
        var["subpathSettings"]["disableCanonical"] = mOptions.subpathSetting.disableCanonical;
        var["subpathSettings"]["compressNeighborSearchKey"] = mOptions.subpathSetting.compressNeighborSearchKey;

        var["subpathSettings"]["knnSearchRadiusMultiplier"] = mOptions.subpathSetting.knnSearchRadiusMultiplier;
        var["subpathSettings"]["knnSearchAdaptiveRadiusType"] = mOptions.subpathSetting.knnSearchAdaptiveRadiusType;
        var["subpathSettings"]["knnIncludeDirectionSearch"] = mOptions.subpathSetting.knnIncludeDirectionSearch;

        var["subpathSettings"]["useMMIS"] = mOptions.subpathSetting.useMMIS;
```
- 這部分代碼將 C++ 程序中的 mOptions 中的 subpathSetting 中的特定設置值傳遞給 shader 中的 subpathSettings 變量，這些值包括：

    - adaptivePrefixLength：指定了自適應前綴長度的設置，可能是移位映射中的一個參數。
    - avoidSpecularPrefixEndVertex：指定了是否避免鏡面前綴結束頂點的設置，可能是移位映射中的一個參數。
    - avoidShortPrefixEndSegment：指定了是否避免短前綴結束段的設置，可能是移位映射中的一個參數。
    - shortSegmentThreshold：指定了短段的閾值，可能是移位映射中的一個參數。
    - suffixSpatialNeighborCount：指定了後綴空間鄰域的數量。
    - suffixSpatialReuseRadius：指定了後綴空間重用的半徑。
    - suffixSpatialReuseRounds：指定了後綴空間重用的輪數。
    - numIntegrationPrefixes：指定了積分前綴的數量。
    - generateCanonicalSuffixForEachPrefix：指定了是否為每個前綴生成標準後綴。
    - suffixTemporalReuse：指定了後綴時間重用的設置。
    - temporalHistoryLength：指定了時間歷史記錄的長度。
    - prefixNeighborSearchRadius：指定了前綴鄰域搜索的半徑。
    - prefixNeighborSearchNeighborCount：指定了前綴鄰域搜索的鄰居數量。
    - finalGatherSuffixCount：指定了最終聚合後綴的數量。
    - useTalbotMISForGather：指定了是否使用 Talbot 多重重要性抽樣進行聚合。
    - nonCanonicalWeightMultiplier：指定了非標準權重的乘數。
    - disableCanonical：指定了是否禁用標準後綴。
    - compressNeighborSearchKey：指定了是否壓縮鄰域搜索鍵。
    - knnSearchRadiusMultiplier：指定了 KNN 搜索半徑的倍增因子。
    - knnSearchAdaptiveRadiusType：指定了 KNN 搜索的自適應半徑類型。
    - knnIncludeDirectionSearch：指定了是否包含方向搜索。
    - useMMIS：指定了是否使用 MMIS（Multiple Importance Sampling with Multiple Importance Sampling）進行計算。

```cpp
        var["minimumPrefixLength"] = mOptions.minimumPrefixLength;
```
- 這行代碼將 C++ 程序中的 mOptions 中的 minimumPrefixLength 值傳遞給 shader 中的 minimumPrefixLength 變量，這個值用於指定最小前綴長度。

```cpp
        int numRounds = mOptions.subpathSetting.suffixSpatialReuseRounds + 1; //include the prefix streaming pass
        numRounds = mOptions.subpathSetting.suffixTemporalReuse ? numRounds + 1 : numRounds;
        var["suffixSpatialRounds"] = numRounds;
        var["pathReservoirs"] = mpScratchReservoirs;
        var["prefixGBuffer"] = mpScratchPrefixGBuffer;
        var["prefixPathReservoirs"] = mpPrefixPathReservoirs;
        var["prefixThroughputs"] = mpPrefixThroughputs;
        var["prefixReservoirs"] = mpPrefixReservoirs;
        float3 worldBoundExtent = mpScene->getSceneBounds().extent();
        var["sceneRadius"] = std::min(worldBoundExtent.x, std::min(worldBoundExtent.y, worldBoundExtent.z));
        
        var["needResetTemporalHistory"] = mResetTemporalReservoirs;
        var["samplesPerPixel"] = mPathTracerParams.samplesPerPixel;
        var["shiftMapping"] = (uint32_t)mOptions.shiftMapping;
    }
```
- 這段代碼的作用如下：

    - numRounds 變數計算了在空間重用迴圈中執行的回合數。如果啟用了時間重用，則在計算回合數時會再增加 1。這個值最後被傳遞到 shader 中的 suffixSpatialRounds 變數。
    - pathReservoirs、prefixGBuffer、prefixPathReservoirs、prefixThroughputs 和 prefixReservoirs 是在 C++ 代碼中定義的一些資源或變數，它們被傳遞到 shader 中，以便 shader 中能夠使用這些資源或變數進行計算。
    - worldBoundExtent 變數用於獲取場景的邊界範圍。然後計算出三個維度中的最小值，並將其傳遞到 shader 中的 sceneRadius 變數中，以提供場景半徑的估算。
    - needResetTemporalHistory 變數將 C++ 程序中的 mResetTemporalReservoirs 值傳遞給 shader 中的 needResetTemporalHistory 變量。
    - samplesPerPixel 變數將 C++ 程序中的 mPathTracerParams.samplesPerPixel 值傳遞給 shader 中的 samplesPerPixel 變量。
    - shiftMapping 變數將 C++ 程序中的 mOptions.shiftMapping 值轉換為 uint32_t 類型，並將其傳遞給 shader 中的 shiftMapping 變量。

```cpp
    void ConditionalReSTIRPass::setPathTracerParams(int useFixedSeed, uint fixedSeed,
    float lodBias, float specularRoughnessThreshold, uint2 frameDim, uint2 screenTiles, uint frameCount, uint seed,
    int samplesPerPixel, int DIMode)
    {
        mPathTracerParams.useFixedSeed = useFixedSeed;
        mPathTracerParams.fixedSeed = fixedSeed;
        mPathTracerParams.lodBias = lodBias;
        mPathTracerParams.specularRoughnessThreshold = specularRoughnessThreshold;
        mPathTracerParams.frameDim = frameDim;
        mPathTracerParams.screenTiles = screenTiles;
        mPathTracerParams.frameCount = frameCount;
        mPathTracerParams.seed = seed;
        mPathTracerParams.samplesPerPixel = samplesPerPixel;
        mPathTracerParams.DIMode = DIMode;
    }
```
- 這是一個設置路徑追踪器參數的函數。該函數接受多個參數，用於配置路徑追踪器的行為。下面是每個參數的作用：

    - useFixedSeed: 一個整數，指示是否使用固定種子來生成隨機數。如果設置為1，則使用固定種子；如果設置為0，則使用每幀不同的種子。
    - fixedSeed: 一個無符號整數，表示固定的種子值。如果 useFixedSeed 設置為1，則路徑追踪器將使用此種子生成隨機數。
    - lodBias: 一個浮點數，表示細節層級的偏移量。
    - specularRoughnessThreshold: 一個浮點數，表示反射率的粗糙閾值。低於此閾值的表面將被視為鏡面反射。
    - frameDim: 一個包含兩個無符號整數的二維向量，表示幀的寬度和高度。
    - screenTiles: 一個包含兩個無符號整數的二維向量，表示屏幕分割的網格數。
    - frameCount: 一個無符號整數，表示當前幀的計數。
    - seed: 一個無符號整數，表示隨機數種子值。
    - samplesPerPixel: 一個整數，表示每個像素的每次渲染中採樣的次數。
    - DIMode: 一個整數，表示深度優化模式的選擇。

- 這個函數將這些參數值設置到 mPathTracerParams 中，以便在路徑追踪器中使用。

```cpp
    void ConditionalReSTIRPass::setOwnerDefines(Program::DefineList defines)
    {
        mOwnerDefines = defines;
    }
```
- 這個函數是用來設置擁有者定義列表的。它接受一個 Program::DefineList 作為參數，並將其存儲在 mOwnerDefines 中。這樣做的目的是為了在 ConditionalReSTIRPass 對象中存儲擁有者定義列表，以供後續使用。

```cpp
    void ConditionalReSTIRPass::setSharedStaticParams(uint32_t samplesPerPixel, uint32_t maxSurfaceBounces, bool useNEE)
    {
        mStaticParams.maxSurfaceBounces = maxSurfaceBounces;
        mStaticParams.useNEE = useNEE;
    }
```
- 這個函數是用來設置共享的靜態參數的。它接受三個參數：samplesPerPixel、maxSurfaceBounces 和 useNEE。然後將這些值分別設置到 mStaticParams 的相應成員變量中，以便在 ConditionalReSTIRPass 對象中進行後續使用。這些參數通常用於控制渲染過程中的一些共享行為，如表面反射的最大反射次數和是否使用下一事件估計（NEE）。

```cpp
    void ConditionalReSTIRPass::createPathTracerBlock()
    {
        auto reflector = mpReflectTypes->getProgram()->getReflector()->getParameterBlock("pathTracer");
        mpPathTracerBlock = ParameterBlock::create(reflector);
    }
```
- 這個函數用於創建路徑追踪器（Path Tracer）的參數塊（Parameter Block）。它首先通過 mpReflectTypes 對象獲取路徑追踪器的程序（Program），然後從該程序的反射器（Reflector）中找到名為 "pathTracer" 的參數塊。接著使用這個參數塊的描述來創建一個新的參數塊 mpPathTracerBlock。這個參數塊通常用於將路徑追踪器所需的參數傳遞給相應的計算着色器（Compute Shader）或其他需要這些參數的地方。

```cpp
    ParameterBlock::SharedPtr ConditionalReSTIRPass::getPathTracerBlock()
    {
        return mpPathTracerBlock;
    }
```
- 這是一個成員函數，返回用於路徑追踪器的參數塊（Parameter Block）。該函數返回 mpPathTracerBlock，這是在之前的 createPathTracerBlock 函數中創建的參數塊。通常，開發人員可以使用這個函數來獲取路徑追踪器的參數塊，以便在需要時設置或修改參數。

```cpp
    void ConditionalReSTIRPass::setReservoirData(const ShaderVar& var) const
    {
        var["pathReservoirs"] = mpReservoirs;
    }
```
- 這個函數是用來設置儲存器數據的。它接受一個 ShaderVar 類型的參數 var，通常是用來與 GPU 中的着色器變量進行交互的。在這個函數中，它將名為 "pathReservoirs" 的着色器變量設置為 mpReservoirs，這個變量可能是儲存路徑追蹤器數據的數組或緩衝區。通常，這種函數用於將 CPU 上的數據傳遞給 GPU，以便在 GPU 上執行相應的計算。

```cpp
    bool ConditionalReSTIRPass::renderUI(Gui::Widgets& widget)
    {
        bool dirty = false;

        if (auto group = widget.group("Performance settings", true))
        {
            mReallocate |= group.checkbox("Use reservoir compression", mOptions.useReservoirCompression);
            mReallocate |= group.dropdown("Retrace Schedule Type", kRetraceScheduleType, reinterpret_cast<uint32_t&>(mOptions.retraceScheduleType));
        }

        if (auto group = widget.group("Subpath reuse", true))
        {
            dirty |= group.var("Num Integration Prefixes", mOptions.subpathSetting.numIntegrationPrefixes, 1, 128);
            dirty |= group.checkbox("Generate Canonical Suffix For Each Prefix", mOptions.subpathSetting.generateCanonicalSuffixForEachPrefix);
            dirty |= group.checkbox("Use MMIS", mOptions.subpathSetting.useMMIS);
            dirty |= group.var("Min Prefix Length", mOptions.minimumPrefixLength, 1u, mStaticParams.maxSurfaceBounces);
            dirty |= group.checkbox("Adaptive Prefix Length", mOptions.subpathSetting.adaptivePrefixLength);
            dirty |= group.checkbox("Avoid Specular Prefix End Vertex", mOptions.subpathSetting.avoidSpecularPrefixEndVertex);
            dirty |= group.checkbox("Avoid Short Prefix End Segment", mOptions.subpathSetting.avoidShortPrefixEndSegment);
            dirty |= group.var("Short Segment Threshold", mOptions.subpathSetting.shortSegmentThreshold, 0.f, 0.1f);

            mReallocate |= group.var("Suffix Spatial Neighbors", mOptions.subpathSetting.suffixSpatialNeighborCount, 1, 8);
            dirty |= group.var("Suffix Spatial Reuse Radius", mOptions.subpathSetting.suffixSpatialReuseRadius, 0.f, 100.f);

            {
                dirty |= group.var("Suffix Reuse rounds", mOptions.subpathSetting.suffixSpatialReuseRounds, 0, 16);
                dirty |= group.checkbox("Suffix Temporal Reuse", mOptions.subpathSetting.suffixTemporalReuse);
                dirty |= group.var("Suffix Temopral History Length", mOptions.subpathSetting.temporalHistoryLength, 0, 100);

                mReallocate |= group.var("Final Gather Suffix Count", mOptions.subpathSetting.finalGatherSuffixCount, 1, 8);
                mReallocate |= group.checkbox("Use Talbot MIS For Gather", mOptions.subpathSetting.useTalbotMISForGather);
                dirty |= group.var("Non-Canonical Weight Multiplier", mOptions.subpathSetting.nonCanonicalWeightMultiplier, 0.f, 100.f);
                dirty |= group.checkbox("Disable Canonical", mOptions.subpathSetting.disableCanonical);

                dirty |= group.var("KNN Search Radius Multiplier", mOptions.subpathSetting.knnSearchRadiusMultiplier);
                dirty |= group.dropdown("KNN Search Adaptive Type", kKNNAdaptiveRadiusType, reinterpret_cast<uint32_t&>(mOptions.subpathSetting.knnSearchAdaptiveRadiusType));
                dirty |= group.checkbox("KNN Inlucde Direction Search For Low Roughness", mOptions.subpathSetting.knnIncludeDirectionSearch);
                if (mOptions.subpathSetting.knnIncludeDirectionSearch)
                {
                    dirty |= group.var("Final Gather Screen Search Radius", mOptions.subpathSetting.prefixNeighborSearchRadius, 0, 100);
                    dirty |= group.var("Final Gather Screen Search Neighbors", mOptions.subpathSetting.prefixNeighborSearchNeighborCount, 0, 100);
                }

                mReallocate |= group.checkbox("Compress Neighbor Search Key", mOptions.subpathSetting.compressNeighborSearchKey);
            }
        }

        if (auto group = widget.group("Shift mapping options", true))
        {
            mRecompile |= group.dropdown("Shift Mapping", kShiftMappingList, reinterpret_cast<uint32_t&>(mOptions.shiftMapping));

            if (mOptions.shiftMapping == ConditionalReSTIR::ShiftMapping::Hybrid)
            {
                dirty |= group.var("Distance Threshold", mOptions.shiftMappingSettings.nearFieldDistanceThreshold);
                dirty |= group.var("Roughness Threshold", mOptions.shiftMappingSettings.specularRoughnessThreshold);
            }
        }

        mReallocate |= widget.checkbox("Temporal Reservoir Update for Dynamic Scenes", mOptions.temporalUpdateForDynamicScene);

        mRecompile |= widget.checkbox("Use Prev Frame Scene Data", mOptions.usePrevFrameSceneData);


        if (auto group = widget.group("Debugging"))
        {
            mpPixelDebug->renderUI(group);
        }

        mRecompile |= mReallocate;
        dirty |= mRecompile;

        if (dirty) mResetTemporalReservoirs = true;

        return dirty;
    }
```
- 這段代碼是用於渲染用戶界面（UI）的。它使用了一個名為 Gui::Widgets 的類，該類可能是用於創建和管理圖形用戶界面元素的工具類。以下是這段代碼的大致功能：

    - 創建一個名為 "Performance settings" 的可展開組，其中包含了一些性能相關的選項，例如是否使用儲存器壓縮和重新追蹤排程類型。
    - 創建一個名為 "Subpath reuse" 的可展開組，其中包含了一些子路徑重用相關的選項，例如整合子路徑的數量、是否生成每個前綴的標準後綴等。
    - 創建一個名為 "Shift mapping options" 的可展開組，其中包含了一些位移映射相關的選項，例如位移映射類型和相關的閾值。
    - 創建一個名為 "Debugging" 的組，用於調試目的。

- 代碼中還包含了一些變量（如 mReallocate、mRecompile 和 mResetTemporalReservoirs），用於跟蹤界面中的更改並做出相應的反應。

- 總的來說，這段代碼負責渲染一個用於設置 ConditionalReSTIRPass 相關選項的用戶界面，並且能夠處理用戶對界面元素的更改。

```cpp
    void ConditionalReSTIRPass::setOptions(const Options& options)
    {
        if (std::memcmp(&options, &mOptions, sizeof(Options)) != 0)
        {
            mOptions = options;
            mRecompile = true;
        }
    }
```
- 這個函數用於設置 ConditionalReSTIRPass 的選項。它接受一個名為 options 的參數，該參數是一個 Options 類型的對象，包含了要設置的選項值。
- 函數首先使用 std::memcmp 函數將新的選項值 options 與當前存儲的選項值 mOptions 進行比較。如果兩者的內容不相等（即存在更改），則將新的選項值複製到 mOptions 中，並將 mRecompile 標記設置為 true，表示需要重新編譯相關的內容以反映這些更改。
- 這樣做的目的是在新的選項值被設置時，僅在發生了實際的更改時才觸發重新編譯相關的內容，以提高效率並減少不必要的操作。

```cpp
    void ConditionalReSTIRPass::beginFrame(RenderContext* pRenderContext, const uint2& frameDim, const uint2& screenTiles, bool needRecompile)
    {
        mRecompile |= needRecompile;

        mFrameDim = frameDim;

        prepareResources(pRenderContext, frameDim, screenTiles);

        mpPixelDebug->beginFrame(pRenderContext, mFrameDim);
    }
```
- 這是 ConditionalReSTIRPass 類的成員函數 beginFrame。它用於開始一個新的渲染幀。

- 此函數接受以下參數：

    - pRenderContext：指向渲染上下文的指針，用於在渲染期間執行渲染命令。
    - frameDim：表示幀尺寸的二維向量。
    - screenTiles：表示屏幕瓦片的二維向量。
    - needRecompile：一個布爾值，表示是否需要重新編譯。

- 在函數內部，它執行以下操作：

    - 將成員變量 mRecompile 設置為 true，如果 needRecompile 為 true，則表示需要重新編譯。
    - 將成員變量 mFrameDim 設置為 frameDim，即設置當前幀的尺寸。
    - 調用 prepareResources 函數，準備渲染所需的資源，並將 frameDim 和 screenTiles 作為參數傳遞給它。
    - 調用 mpPixelDebug 對象的 beginFrame 函數，開始新的調試幀，並將 pRenderContext 和 mFrameDim 作為參數傳遞給它。

- 總之，這個函數是用於初始化渲染幀所需的資源，並準備進行渲染操作。

```cpp
    void ConditionalReSTIRPass::endFrame(RenderContext* pRenderContext)
    {
        mFrameIndex++;

        // Swap reservoirs.
        if (!mpScene->freeze)
        {
            std::swap(mpPrefixReservoirs, mpPrevPrefixReservoirs);
            std::swap(mpReservoirs, mpPrevReservoirs);
            std::swap(mpPrefixGBuffer, mpPrevPrefixGBuffer);
        }

        mpPixelDebug->endFrame(pRenderContext);
    }
```
- 這是 ConditionalReSTIRPass 類的成員函數 endFrame。它用於結束當前渲染幀。

- 此函數接受以下參數：

    - pRenderContext：指向渲染上下文的指針，用於在渲染期間執行渲染命令。

- 在函數內部，它執行以下操作：

    - 將成員變量 mFrameIndex 增加1，表示當前幀的索引。
    - 如果場景不是被凍結的，則交換前綴和後綴的資源，包括前綴和後綴的蓄水池 (mpPrefixReservoirs 和 mpPrevPrefixReservoirs)，以及前綴和後綴的 GBuffer (mpPrefixGBuffer 和 mpPrevPrefixGBuffer)。
    - 調用 mpPixelDebug 對象的 endFrame 函數，結束當前的調試幀，並將 pRenderContext 作為參數傳遞給它。

- 總之，這個函數是用於結束當前渲染幀，進行必要的清理操作，並準備開始下一個渲染幀。

```cpp
    void ConditionalReSTIRPass::createOrDestroyBuffer(Buffer::SharedPtr& pBuffer, std::string reflectVarName, int requiredElementCount, bool keepCondition)
    {
        if (keepCondition && (mReallocate || !pBuffer || pBuffer->getElementCount() != requiredElementCount))
            pBuffer = Buffer::createStructured(mpReflectTypes[reflectVarName], requiredElementCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        if (!keepCondition) pBuffer = nullptr;
    }
```
- 這是 ConditionalReSTIRPass 類的成員函數 createOrDestroyBuffer。它用於創建或銷毀緩衝區，根據指定的條件和參數。

- 此函數接受以下參數：

    - pBuffer：要操作的緩衝區的引用。此引用將根據函數的操作而被修改。
    - reflectVarName：用於創建緩衝區的反射變量的名稱。
    - requiredElementCount：所需的緩衝區元素數量。
    - keepCondition：一個條件，如果為 true，則保留緩衝區，否則銷毀緩衝區。

- 在函數內部，它執行以下操作：

    - 如果 keepCondition 為真且滿足以下任一條件之一：
        - mReallocate 為真，表示需要重新分配緩衝區。
        - pBuffer 為空指針，表示緩衝區尚未創建。
        - pBuffer 的元素數量與 requiredElementCount 不相等，需要重新分配緩衝區。
          則創建一個新的結構化緩衝區，其元素數量為 requiredElementCount，並根據反射變量的名稱和綁定標誌進行綁定。如果緩衝區已經存在，則其內容將被清空。
    - 如果 keepCondition 為假，則銷毀緩衝區，將其設置為空指針。

- 總之，這個函數根據給定的條件和參數創建或銷毀緩衝區。

```cpp
    void ConditionalReSTIRPass::createOrDestroyBufferWithCounter(Buffer::SharedPtr& pBuffer, std::string reflectVarName, int requiredElementCount, bool keepCondition)
    {
        if (keepCondition && (mReallocate || !pBuffer || pBuffer->getElementCount() != requiredElementCount))
            pBuffer = Buffer::createStructured(mpReflectTypes[reflectVarName], requiredElementCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, true);
        if (!keepCondition) pBuffer = nullptr;
    }
```
- 這是 ConditionalReSTIRPass 類的另一個成員函數 createOrDestroyBufferWithCounter。與前一個函數相比，這個函數創建的是帶有計數器的緩衝區。

- 這個函數與 createOrDestroyBuffer 的操作類似，唯一的區別是創建的緩衝區帶有計數器。計數器可以在 GPU 上跟踪對緩衝區的讀寫操作，對於某些操作，特別是需要原子操作的情況，使用計數器可以更有效地實現。

- 因此，這個函數的操作與 createOrDestroyBuffer 相同，唯一的區別是創建的緩衝區帶有計數器。

```cpp
    void ConditionalReSTIRPass::createOrDestroyBufferNoReallocate(Buffer::SharedPtr& pBuffer, std::string reflectVarName, int requiredElementCount, bool keepCondition)
    {
        if (keepCondition && (!pBuffer || pBuffer->getElementCount() != requiredElementCount))
            pBuffer = Buffer::createStructured(mpReflectTypes[reflectVarName], requiredElementCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        if (!keepCondition) pBuffer = nullptr;
    }
```
- 這是 ConditionalReSTIRPass 類的另一個成員函數 createOrDestroyBufferNoReallocate。

- 這個函數與 createOrDestroyBuffer 的作用類似，但它不會在 mReallocate 為真或緩衝區不存在或元素數量不符合要求時重新分配緩衝區。它僅在 keepCondition 為真且緩衝區不存在或元素數量不符合要求時創建新的緩衝區。

- 這個函數的目的是為了在不需要重新分配緩衝區的情況下，根據條件創建或銷毀緩衝區。

```cpp
    void ConditionalReSTIRPass::createOrDestroyBufferWithCounterNoReallocate(Buffer::SharedPtr& pBuffer, std::string reflectVarName, int requiredElementCount, bool keepCondition)
    {
        if (keepCondition && (!pBuffer || pBuffer->getElementCount() != requiredElementCount))
            pBuffer = Buffer::createStructured(mpReflectTypes[reflectVarName], requiredElementCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, true);
        if (!keepCondition) pBuffer = nullptr;
    }
```
- 這是 ConditionalReSTIRPass 類的另一個成員函數 createOrDestroyBufferWithCounterNoReallocate。

- 這個函數與 createOrDestroyBufferNoReallocate 的作用類似，但它在創建緩衝區時使用了帶有計數器的緩衝區，這意味著可以使用原子操作來對緩衝區中的元素進行計數。

- 同樣，這個函數也僅在 keepCondition 為真且緩衝區不存在或元素數量不符合要求時創建新的緩衝區。

```cpp
    void ConditionalReSTIRPass::createOrDestroyRawBuffer(Buffer::SharedPtr& pBuffer, size_t requiredSize, bool keepCondition)
    {
        if (keepCondition && (mReallocate || !pBuffer || pBuffer->getSize() != requiredSize))
            pBuffer = Buffer::create(requiredSize, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr);
        if (!keepCondition) pBuffer = nullptr;
    }
```
- 這個函數是 ConditionalReSTIRPass 類中的一個成員函數 createOrDestroyRawBuffer。

- 該函數的作用是根據給定的條件來創建或銷毀一個原始的緩衝區（Raw Buffer）。

- 如果 keepCondition 為真並且滿足以下條件之一，則會創建一個新的緩衝區：

    - mReallocate 為真（這通常表示需要重新分配資源）。
    - 緩衝區 pBuffer 為空指針。
    - 緩衝區 pBuffer 的大小不等於所需的大小 requiredSize。

- 如果 keepCondition 為假，則會將緩衝區 pBuffer 設置為空指針，即銷毀緩衝區。

- 該函數用於在需要時動態創建或銷毀原始緩衝區，以滿足特定的條件和需求。

```cpp
    void ConditionalReSTIRPass::prepareResources(RenderContext* pRenderContext, const uint2& frameDim, const uint2& screenTiles)
    {
        // disable hybrid shift, temporal
        if (mReallocate && mpReservoirs) updatePrograms();
        // Create screen sized buffers.
        uint32_t tileCount = screenTiles.x * screenTiles.y;
        const uint32_t elementCount = tileCount * kScreenTileDim.x * kScreenTileDim.y;

        // getting correct struct sizes when initializing
        if (!mpReservoirs)
        {
            Program::DefineList defines;
            defines.add(getDefines());
            Program::TypeConformanceList typeConformances;
            // Scene-specific configuration.
            typeConformances.add(mpScene->getTypeConformances());
            Program::Desc baseDesc;
            baseDesc.addShaderModules(mpScene->getShaderModules());
            baseDesc.addTypeConformances(typeConformances);
            baseDesc.setShaderModel(kShaderModel);
            createComputePass(mpReflectTypes, kReflectTypesFile, defines, baseDesc);
        }

        createOrDestroyBuffer(mpReservoirs, "pathReservoirs", elementCount);
        createOrDestroyBuffer(mpPrevReservoirs, "pathReservoirs", elementCount);
        createOrDestroyBuffer(mpScratchReservoirs, "pathReservoirs", elementCount);
        createOrDestroyBuffer(mpPrefixPathReservoirs, "prefixPathReservoirs", elementCount);
        createOrDestroyBuffer(mpPrefixThroughputs, "prefixThroughputs", elementCount);

        createOrDestroyBuffer(mpPrevSuffixReservoirs, "pathReservoirs", elementCount);
        createOrDestroyBuffer(mpTempReservoirs, "pathReservoirs", elementCount, mpScene->freeze);
        createOrDestroyBuffer(mpNeighborValidMaskBuffer, "neighborValidMask", elementCount);

        // for hybrid shift workload compaction
        int maxNeighborCount = std::max( mOptions.subpathSetting.finalGatherSuffixCount, mOptions.subpathSetting.suffixSpatialNeighborCount);
        const uint32_t talbotPathCount = elementCount * (mOptions.subpathSetting.useTalbotMISForGather ? mOptions.subpathSetting.finalGatherSuffixCount * (mOptions.subpathSetting.finalGatherSuffixCount + 1) : 0);
        const uint32_t pathCount = std::max(talbotPathCount, elementCount * 2 * maxNeighborCount);

        createOrDestroyRawBuffer(mpWorkload, pathCount * sizeof(uint32_t), mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact);
        createOrDestroyRawBuffer(mpWorkloadExtra, pathCount * sizeof(uint32_t), mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact && mOptions.subpathSetting.useTalbotMISForGather);

        createOrDestroyRawBuffer(mpCounter, sizeof(uint32_t), mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact);

        //4*1024*1024*1024 / 48 (rcReconnectionData size at compress reservoir and sd_optim), and round to nearest 10^7 (somehow using originla number causes crash)
        createOrDestroyBuffer(mpReconnectionDataBuffer, "reconnectionDataBuffer", std::min(80000000u, pathCount));
        createOrDestroyBuffer(mpRcBufferOffsets, "rcBufferOffsets", pathCount);

        const uint32_t maxVertexCount = elementCount * (mStaticParams.maxSurfaceBounces + 1);

        createOrDestroyBuffer(mpPrefixGBuffer, "prefixGBuffer", elementCount);
        createOrDestroyBuffer(mpPrevPrefixGBuffer, "prefixGBuffer", elementCount);
        createOrDestroyBuffer(mpFinalGatherSearchKeys, "prefixSearchKeys", elementCount);

        createOrDestroyBuffer(mpPrefixReservoirs, "prefixReservoirs", elementCount);
        createOrDestroyBuffer(mpPrevPrefixReservoirs, "prefixReservoirs", elementCount);

        createOrDestroyBuffer(mpScratchPrefixGBuffer, "prefixGBuffer", elementCount);

        createOrDestroyBuffer(mpFoundNeighborPixels, "foundNeighborPixels", mOptions.subpathSetting.finalGatherSuffixCount * elementCount);

        createOrDestroyBufferWithCounterNoReallocate(mpSearchPointBoundingBoxBuffer, "searchPointBoundingBoxBuffer", frameDim.x * frameDim.y);
        createOrDestroyBufferNoReallocate(mpPrefixL2LengthBuffer, "prefixL2LengthBuffer", frameDim.x * frameDim.y);

        if (!mpTemporalVBuffer || mpTemporalVBuffer->getHeight() != frameDim.y || mpTemporalVBuffer->getWidth() != frameDim.x)
        {
            mpTemporalVBuffer = Texture::create2D(frameDim.x, frameDim.y, mpScene->getHitInfo().getFormat(), 1, 1);
        }

        mReallocate = false;
    }
```
- 這段代碼是 ConditionalReSTIRPass 類中的 prepareResources 函數。

- 這個函數主要負責準備所需的資源，以便在渲染時使用。以下是函數中的一些主要步驟：

    - 如果需要重新分配資源 mReallocate 為真且 mpReservoirs 不為空，則調用 updatePrograms() 函數。
    - 創建與銷毀一些與螢幕大小相關的緩衝區，包括 mpReservoirs、mpPrevReservoirs、mpScratchReservoirs 等。
    - 根據設置的條件和需求，創建或銷毀一些其他緩衝區，如 mpNeighborValidMaskBuffer、mpWorkload、mpCounter 等。
    - 創建或銷毀原始緩衝區，如 mpReconnectionDataBuffer、mpRcBufferOffsets 等。
    - 創建或銷毀與圖像尺寸相關的緩衝區，如 mpPrefixGBuffer、mpPrevPrefixGBuffer、mpFinalGatherSearchKeys 等。
    - 最後，檢查是否需要重新分配資源，並將 mReallocate 設置為 false，以確保下一次函數調用時不會再次重新分配資源。

- 總的來說，這個函數確保所需的資源在渲染前都已經準備好，以確保順利執行後續的渲染操作。

```cpp
    void ConditionalReSTIRPass::updatePrograms()
    {
         if (!mRecompile) return;

         Program::DefineList commonDefines;

         commonDefines.add(getDefines());
         commonDefines.add(mpScene->getSceneDefines());
         commonDefines.add(mOwnerDefines);

         Program::TypeConformanceList typeConformances;
         // Scene-specific configuration.
         typeConformances.add(mpScene->getTypeConformances());

         Program::Desc baseDesc;
         baseDesc.addShaderModules(mpScene->getShaderModules());
         baseDesc.addTypeConformances(typeConformances);
         baseDesc.setShaderModel(kShaderModel);

         Program::DefineList defines = commonDefines;
         defines.add("NEIGHBOR_OFFSET_COUNT", std::to_string(mpNeighborOffsets->getWidth()));

         createComputePass(mpReflectTypes, kReflectTypesFile, defines, baseDesc);
         createComputePass(mpPrefixResampling, kPrefixResampling, defines, baseDesc);
         createComputePass(mpTraceNewSuffixes, kTraceNewSuffixes, defines, baseDesc);
         createComputePass(mpTraceNewPrefixes, kTraceNewPrefixes, defines, baseDesc);
         createComputePass(mpPrefixNeighborSearch, kPrefixNeighborSearch, defines, baseDesc);
         createComputePass(mpSuffixSpatialResampling, kSuffixSpatialResamplingFile, defines, baseDesc, "spatial");
         createComputePass(mpSuffixTemporalResampling, kSuffixTemporalResamplingFile, defines, baseDesc, "temporal");
         createComputePass(mpSuffixResampling, kSuffixResamplingFile, defines, baseDesc, "gather");
         createComputePass(mpPrefixRetrace, kPrefixRetraceFile, defines, baseDesc);
         createComputePass(mpPrefixProduceRetraceWorkload, kPrefixProduceRetraceWorkload, defines, baseDesc);
         createComputePass(mpSuffixRetrace, kSuffixRetraceFile, defines, baseDesc);
         createComputePass(mpSuffixProduceRetraceWorkload, kSuffixProduceRetraceWorkload, defines, baseDesc);
         createComputePass(mpSuffixRetraceTalbot, kSuffixRetraceTalbotFile, defines, baseDesc);
         createComputePass(mpSuffixProduceRetraceTalbotWorkload, kSuffixProduceRetraceTalbotWorkload, defines, baseDesc);

         mRecompile = false;
         mResetTemporalReservoirs = true;
    }
```
- 這段代碼是 ConditionalReSTIRPass 類中的 updatePrograms 函數。

- 這個函數主要負責更新渲染所需的程序（Programs）。以下是函數中的一些主要步驟：

    - 如果不需要重新編譯（mRecompile 為假），則直接返回，不執行後續操作。
    - 構建包含共享定義（Defines）的列表 commonDefines，其中包括從 getDefines()、mpScene->getSceneDefines() 和 mOwnerDefines 中獲取的定義。
    - 構建類型相容性列表 typeConformances，其中包括從 mpScene->getTypeConformances() 中獲取的類型相容性。
    - 構建程序描述 baseDesc，其中包括從 mpScene->getShaderModules() 和 typeConformances 中獲取的信息，並設置着色器模型為 kShaderModel。
    - 構建程序定義列表 defines，將 commonDefines 與 "NEIGHBOR_OFFSET_COUNT" 的值（來自 mpNeighborOffsets->getWidth()）結合起來。
    - 使用 createComputePass 函數創建一系列的計算程序（Compute Pass），分別為不同的渲染階段，包括重新採樣、追踪、回溯等。
    - 最後，將 mRecompile 和 mResetTemporalReservoirs 設置為 false，以確保下一次函數調用時不會再次重新編譯程序，並標記需要重置時間蓄水池。

- 總的來說，這個函數確保渲染所需的程序已經更新並準備好在後續的渲染操作中使用。

---
- 這段代碼是 ConditionalReSTIRPass 類中的 suffixResamplingPass 函數。

- 這個函數主要負責後綴（Suffix）重新採樣的操作。以下是函數中的一些主要步驟：

    - 根據需要設置是否具有時間重用（Temporal Reuse）。
    - 綁定後綴重新採樣所需的變數和資源。
    - 分別執行空間重新採樣、時間重新採樣和後綴重新採樣。
    - 如果使用壓縮的重新採樣方案，則生成壓縮後綴重新採樣的工作量。
    - 如果使用達爾伯特（Talbot）多重重要性採樣（MIS）來進行後綴收集，則執行相應的工作量生成和後綴重新採樣。
    - 如果啟用自適應前綴長度，則根據配置執行前綴重新採樣和前綴重新追蹤。
    - 最後準備時間數據，包括拷貝緩衝區、更新相機屬性等操作。

- 總的來說，這個函數是後綴重新採樣的核心，負責生成、重建和整合後綴路徑，並準備相應的數據以供後續使用。

```cpp
    void ConditionalReSTIRPass::suffixResamplingPass(
        RenderContext* pRenderContext,
        const Texture::SharedPtr& pVBuffer,
        const Texture::SharedPtr& pMotionVectors,
        const Texture::SharedPtr& pOutputColor
    )
    {
        FALCOR_PROFILE("SuffixResampling");

        bool hasTemporalReuse = mOptions.subpathSetting.suffixTemporalReuse;
        // if we have no temporal history, skip the first round (set suffixTemporalReuse in CB to false temporarily)
        mOptions.subpathSetting.suffixTemporalReuse = (mResetTemporalReservoirs) ? false : mOptions.subpathSetting.suffixTemporalReuse;

        ShaderVar presamplingVar = bindSuffixResamplingVars(pRenderContext, mpPrefixResampling, "gPrefixResampling", pVBuffer, pMotionVectors, true, true);
        presamplingVar["prevCameraU"] = mPrevCameraU;
        presamplingVar["prevCameraV"] = mPrevCameraV;
        presamplingVar["prevCameraW"] = mPrevCameraW;
        presamplingVar["prevJitterX"] = mPrevJitterX;
        presamplingVar["prevJitterY"] = mPrevJitterY;
        presamplingVar["prefixReservoirs"] = mpPrefixReservoirs;
        presamplingVar["prevPrefixReservoirs"] = mpPrevPrefixReservoirs;
        presamplingVar["rcBufferOffsets"] = mpRcBufferOffsets;
        presamplingVar["reconnectionDataBuffer"] = mpReconnectionDataBuffer;

        ShaderVar spatialVar = bindSuffixResamplingVars(pRenderContext, mpSuffixSpatialResampling, "gSuffixResampling", pVBuffer, pMotionVectors, true, false);
        spatialVar["outColor"] = pOutputColor;
        spatialVar["rcBufferOffsets"] = mpRcBufferOffsets;
        spatialVar["reconnectionDataBuffer"] = mpReconnectionDataBuffer;
        spatialVar["foundNeighborPixels"] = mpFoundNeighborPixels;

        ShaderVar temporalVar = bindSuffixResamplingVars(pRenderContext, mpSuffixTemporalResampling, "gSuffixResampling", pVBuffer, pMotionVectors, true, false);
        temporalVar["outColor"] = pOutputColor;
        temporalVar["rcBufferOffsets"] = mpRcBufferOffsets;
        temporalVar["reconnectionDataBuffer"] = mpReconnectionDataBuffer;
        temporalVar["foundNeighborPixels"] = mpFoundNeighborPixels;

        temporalVar["prevCameraU"] = mPrevCameraU;
        temporalVar["prevCameraV"] = mPrevCameraV;
        temporalVar["prevCameraW"] = mPrevCameraW;
        temporalVar["prevJitterX"] = mPrevJitterX;
        temporalVar["prevJitterY"] = mPrevJitterY;

        ShaderVar prefixVar = bindSuffixResamplingVars(pRenderContext, mpSuffixResampling, "gSuffixResampling", pVBuffer, pMotionVectors, true, true);
        prefixVar["outColor"] = pOutputColor;
        prefixVar["rcBufferOffsets"] = mpRcBufferOffsets;
        prefixVar["reconnectionDataBuffer"] = mpReconnectionDataBuffer;
        prefixVar["foundNeighborPixels"] = mpFoundNeighborPixels;

        ShaderVar workloadVar;
        if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact)
        {
            workloadVar = bindSuffixResamplingVars(
                pRenderContext, mpSuffixProduceRetraceWorkload, "gPathGenerator", pVBuffer, pMotionVectors, false, false
            );
            workloadVar["queue"]["counter"] = mpCounter;
            workloadVar["queue"]["workload"] = mpWorkload;
            workloadVar["foundNeighborPixels"] = mpFoundNeighborPixels;
        }

        ShaderVar retraceVar = bindSuffixResamplingVars(
            pRenderContext, mpSuffixRetrace, "gSuffixPathRetrace", pVBuffer, pMotionVectors, true, false
        );
        retraceVar["reconnectionDataBuffer"] = mpReconnectionDataBuffer;
        retraceVar["rcBufferOffsets"] = mpRcBufferOffsets;
        retraceVar["queue"]["counter"] = mpCounter;
        retraceVar["queue"]["workload"] = mpWorkload;
        retraceVar["foundNeighborPixels"] = mpFoundNeighborPixels;

        ShaderVar workloadVarTalbot;
        if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact && mOptions.subpathSetting.useTalbotMISForGather)
        {
            workloadVarTalbot = bindSuffixResamplingVars(
                pRenderContext, mpSuffixProduceRetraceTalbotWorkload, "gPathGenerator", pVBuffer, pMotionVectors, false, false
            );
            workloadVarTalbot["queue"]["counter"] = mpCounter;
            workloadVarTalbot["queue"]["workload"] = mpWorkload;
            workloadVarTalbot["queue"]["workloadExtra"] = mpWorkloadExtra;
            workloadVarTalbot["foundNeighborPixels"] = mpFoundNeighborPixels;
        }

        ShaderVar retraceVarTalbot = bindSuffixResamplingVars(
            pRenderContext, mpSuffixRetraceTalbot, "gSuffixPathRetrace", pVBuffer, pMotionVectors, true, false
        );
        retraceVarTalbot["reconnectionDataBuffer"] = mpReconnectionDataBuffer;
        retraceVarTalbot["rcBufferOffsets"] = mpRcBufferOffsets;
        retraceVarTalbot["queue"]["counter"] = mpCounter;
        retraceVarTalbot["queue"]["workload"] = mpWorkload;
        retraceVarTalbot["queue"]["workloadExtra"] = mpWorkloadExtra;
        retraceVarTalbot["foundNeighborPixels"] = mpFoundNeighborPixels;

        ShaderVar prefixWorkloadVar;
        if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact)
        {
            prefixWorkloadVar = bindPrefixResamplingVars(
                pRenderContext, mpPrefixProduceRetraceWorkload, "gPathGenerator", pVBuffer, pMotionVectors, false
            );
            prefixWorkloadVar["queue"]["counter"] = mpCounter;
            prefixWorkloadVar["queue"]["workload"] = mpWorkload;
        }

        ShaderVar prefixRetraceVar = bindPrefixResamplingVars(
            pRenderContext, mpPrefixRetrace, "gPrefixPathRetrace", pVBuffer, pMotionVectors, true
        );
        prefixRetraceVar["reconnectionDataBuffer"] = mpReconnectionDataBuffer;
        prefixRetraceVar["rcBufferOffsets"] = mpRcBufferOffsets;
        prefixRetraceVar["queue"]["counter"] = mpCounter;
        prefixRetraceVar["queue"]["workload"] = mpWorkload;
        prefixRetraceVar["prefixReservoirs"] = mpPrefixReservoirs;
        prefixRetraceVar["prevPrefixReservoirs"] = mpPrevPrefixReservoirs;
        prefixRetraceVar["prefixTotalLengthBuffer"] = mpPrefixL2LengthBuffer; // abuse the storage for this

        // try to re-bind the correct value for a term used to offset RNG
        int numRounds = mOptions.subpathSetting.suffixSpatialReuseRounds;
        int numRoundsForComputeRNG = numRounds + 1 + (hasTemporalReuse ? 1 : 0);
        spatialVar["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;
        temporalVar["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;
        prefixVar["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;

        if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact)
        {
            workloadVar["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;
            prefixWorkloadVar["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;
            if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact && mOptions.subpathSetting.useTalbotMISForGather)
                workloadVarTalbot["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;
        }
        retraceVar["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;
        prefixRetraceVar["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;
        if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact && mOptions.subpathSetting.useTalbotMISForGather)
            retraceVarTalbot["restir"]["suffixSpatialRounds"] = numRoundsForComputeRNG;

        if (mOptions.subpathSetting.adaptivePrefixLength)
        {
            if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact)
            {
                FALCOR_PROFILE("ProducePrefixWorkload");

                if (mpCounter)
                    pRenderContext->clearUAV(mpCounter->getUAV().get(), uint4(0));

                prefixWorkloadVar["prevReservoirs"] = mpPrevSuffixReservoirs;
                const uint32_t tileSize = kScreenTileDim.x * kScreenTileDim.y;
                mpPrefixProduceRetraceWorkload->execute(
                    pRenderContext, mPathTracerParams.screenTiles.x * tileSize, mPathTracerParams.screenTiles.y, 1
                );
            }

            {
                FALCOR_PROFILE("PrefixRetrace");
                prefixRetraceVar["prevReservoirs"] = mpPrevSuffixReservoirs;

                mpPrefixRetrace->execute(pRenderContext,
                    mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Naive ? mFrameDim.x : 2 * mFrameDim.x * mFrameDim.y,
                    mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Naive ? mFrameDim.y : 1, 1);
            }
        }

        {
            FALCOR_PROFILE("PrefixResampling");

            presamplingVar["reservoirs"] = mpReservoirs;
            presamplingVar["prevReservoirs"] = mpPrevSuffixReservoirs;
            presamplingVar["prefixSearchKeys"] = mpFinalGatherSearchKeys;
            presamplingVar["searchPointBoundingBoxBuffer"] = mpSearchPointBoundingBoxBuffer;
            presamplingVar["prefixTotalLengthBuffer"] = mpPrefixL2LengthBuffer;
            presamplingVar["screenSpacePixelSpreadAngle"] = mpScene->getCamera()->computeScreenSpacePixelSpreadAngle(mFrameDim.y);

            mpPrefixResampling->execute(
                pRenderContext, mFrameDim.x, mFrameDim.y, 1
            );
        }

        if (!mpSearchASBuilder)
        {
            mpSearchASBuilder = BoundingBoxAccelerationStructureBuilder::Create(mpSearchPointBoundingBoxBuffer);
        }

        if (!mResetTemporalReservoirs)
        {
            FALCOR_PROFILE("BuildSearchAS");
            uint numSearchPoints = mFrameDim.x * mFrameDim.y;
            mpSearchASBuilder->BuildAS(pRenderContext, numSearchPoints, 1);
        }

        // trace an additional path
        {
            FALCOR_PROFILE("TraceNewSuffixes");

            // Bind global resources.
            auto var = mpTraceNewSuffixes->getRootVar();
            mpScene->setRaytracingShaderData(pRenderContext, var);
            mpPixelDebug->prepareProgram(mpTraceNewSuffixes->getProgram(), var);
            mpPixelStats->prepareProgram(mpTraceNewSuffixes->getProgram(), mpTraceNewSuffixes->getRootVar());

            // Bind the path tracer.
            var["gPathTracer"] = mpPathTracerBlock;
            var["gScheduler"]["prefixGbuffer"] = mpPrefixGBuffer;
            var["gScheduler"]["pathReservoirs"] = mpReservoirs;
            // Full screen dispatch.
            mpTraceNewSuffixes->execute(pRenderContext, mFrameDim.x, mFrameDim.y, 1);
        }

        int numLevels = 1;

        for (int iter = 0; iter < numLevels; iter++)
        {
            int numRounds = mOptions.subpathSetting.suffixSpatialReuseRounds;
            // the actual rounds used
            numRounds = mOptions.subpathSetting.suffixTemporalReuse ? numRounds + 1 : numRounds;

            for (int i = 0; i < numRounds; i++)
            {
                bool isCurrentPassTemporal = mOptions.subpathSetting.suffixTemporalReuse && i == 0;
                Buffer::SharedPtr& pPrevSuffixReservoirs = (isCurrentPassTemporal || !mpScene->freeze) ? mpPrevSuffixReservoirs : mpTempReservoirs;

                if (!isCurrentPassTemporal)
                {
                    std::swap(mpReservoirs, pPrevSuffixReservoirs);
                }

                if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact)
                {
                    FALCOR_PROFILE(isCurrentPassTemporal ? "TemporalSuffixProduceRetraceWorkload" : "SpatialSuffixProduceRetraceWorkload");

                    if (mpCounter)
                        pRenderContext->clearUAV(mpCounter->getUAV().get(), uint4(0));

                    workloadVar["reservoirs"] = mpReservoirs;
                    workloadVar["prevReservoirs"] = pPrevSuffixReservoirs;
                    workloadVar["suffixReuseRoundId"] = i;
                    workloadVar["curPrefixLength"] = numLevels - iter;

                    const uint32_t tileSize = kScreenTileDim.x * kScreenTileDim.y;
                    mpSuffixProduceRetraceWorkload->execute(
                        pRenderContext, mPathTracerParams.screenTiles.x * tileSize, mPathTracerParams.screenTiles.y, 1
                    );
                }

                {
                    FALCOR_PROFILE(isCurrentPassTemporal ? "TemporalSuffixRetrace" : "SpatialSuffixRetrace");

                    retraceVar["reservoirs"] = mpReservoirs;
                    retraceVar["prevReservoirs"] = pPrevSuffixReservoirs;
                    retraceVar["suffixReuseRoundId"] = i;

                    if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Naive)
                        mpSuffixRetrace->execute(pRenderContext, mFrameDim.x, mFrameDim.y, 1);
                    else
                        mpSuffixRetrace->execute(pRenderContext, 2 * (isCurrentPassTemporal ? 1 : mOptions.subpathSetting.suffixSpatialNeighborCount) * mFrameDim.x * mFrameDim.y, 1, 1);
                }


                {
                    FALCOR_PROFILE(isCurrentPassTemporal ? "TemporalSuffixResampling" : "SpatialSuffixResampling");

                    ShaderVar& tempVar = isCurrentPassTemporal ? temporalVar : spatialVar;
                    ComputePass::SharedPtr& tempPass = isCurrentPassTemporal ? mpSuffixTemporalResampling : mpSuffixSpatialResampling;

                    tempVar["reservoirs"] = mpReservoirs;
                    tempVar["prevReservoirs"] = pPrevSuffixReservoirs;
                    tempVar["suffixReuseRoundId"] = i;
                    tempVar["curPrefixLength"] = numLevels - iter;
                    tempVar["vbuffer"] = pVBuffer;

                    tempPass->execute(pRenderContext, mFrameDim.x, mFrameDim.y, 1);
                }
            }

            mOptions.subpathSetting.suffixTemporalReuse = hasTemporalReuse;


            //  generate multiple suffixes
            Buffer::SharedPtr& pPrevSuffixReservoirs = !mpScene->freeze ? mpPrevSuffixReservoirs : mpTempReservoirs;
            std::swap(mpReservoirs, pPrevSuffixReservoirs);

            for (int integrationPrefixId = 0; integrationPrefixId < mOptions.subpathSetting.numIntegrationPrefixes; integrationPrefixId++)
            {
                bool hasCanonicalSuffix =
                    mOptions.subpathSetting.generateCanonicalSuffixForEachPrefix ? true : integrationPrefixId == 0;

                // we borrow the prefix of integrationPrefixId 0 from before
                {
                    // trace new prefixes
                    FALCOR_PROFILE("TraceNewPrefixes");

                    // Bind global resources.
                    auto var = mpTraceNewPrefixes->getRootVar();
                    mpScene->setRaytracingShaderData(pRenderContext, var);
                    mpPixelDebug->prepareProgram(mpTraceNewPrefixes->getProgram(), var);
                    mpPixelStats->prepareProgram(mpTraceNewPrefixes->getProgram(), var);
                    // Bind the path tracer.
                    var["gPathTracer"] = mpPathTracerBlock;
                    var["gScheduler"]["integrationPrefixId"] = integrationPrefixId;
                    var["gScheduler"]["shouldGenerateSuffix"] = hasCanonicalSuffix;
                    // Full screen dispatch.
                    mpTraceNewPrefixes->execute(pRenderContext, mFrameDim.x, mFrameDim.y, 1);
                }

                // stream prefixes
                {
                    FALCOR_PROFILE("FinalGather");

                    {
                        FALCOR_PROFILE("FinalGatherNeighborSearch");

                        mpPrefixNeighborSearch["gScene"] = mpScene->getParameterBlock();
                        auto rootVar = mpPrefixNeighborSearch->getRootVar();
                        mpPixelDebug->prepareProgram(mpPrefixNeighborSearch->getProgram(), rootVar);
                        auto var = rootVar["CB"]["gPrefixNeighborSearch"];

                        var["neighborOffsets"] = mpNeighborOffsets;
                        var["motionVectors"] = pMotionVectors;
                        var["params"].setBlob(mPathTracerParams);
                        setShaderData(var["restir"]);
                        var["prefixGBuffer"] = mpScratchPrefixGBuffer;
                        var["prevPrefixGBuffer"] = mpPrefixGBuffer;
                        var["foundNeighborPixels"] = mpFoundNeighborPixels;
                        var["integrationPrefixId"] = integrationPrefixId;
                        var["prefixSearchKeys"] = mpFinalGatherSearchKeys;
                        var["hasSearchPointAS"] = !mResetTemporalReservoirs;
                        var["searchPointBoundingBoxBuffer"] = mpSearchPointBoundingBoxBuffer;

                        if (mpSearchASBuilder && !mResetTemporalReservoirs)
                            mpSearchASBuilder->SetRaytracingShaderData(var, "gSearchPointAS", 1u);

                        mpPrefixNeighborSearch->execute(pRenderContext, mFrameDim.x, mFrameDim.y, 1);
                    }

                    ComputePass::SharedPtr pFinalGatherRetraceProduceWorkload = mOptions.subpathSetting.useTalbotMISForGather ?
                        mpSuffixProduceRetraceTalbotWorkload : mpSuffixProduceRetraceWorkload;

                    if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Compact)
                    {
                        FALCOR_PROFILE("FinalGatherProduceRetraceWorkload");

                        if (mpCounter)
                            pRenderContext->clearUAV(mpCounter->getUAV().get(), uint4(0));

                        ShaderVar& var = mOptions.subpathSetting.useTalbotMISForGather ? workloadVarTalbot : workloadVar;

                        var["prevReservoirs"] = pPrevSuffixReservoirs;
                        var["suffixReuseRoundId"] = -1;
                        var["integrationPrefixId"] = integrationPrefixId;

                        const uint32_t tileSize = kScreenTileDim.x * kScreenTileDim.y;
                        pFinalGatherRetraceProduceWorkload->execute(
                            pRenderContext, mPathTracerParams.screenTiles.x * tileSize, mPathTracerParams.screenTiles.y, 1
                        );
                    }

                    ComputePass::SharedPtr pSuffixRetrace = mOptions.subpathSetting.useTalbotMISForGather ?
                        mpSuffixRetraceTalbot: mpSuffixRetrace;

                    {
                        FALCOR_PROFILE("FinalGatherSuffixRetrace");

                        ShaderVar& var = mOptions.subpathSetting.useTalbotMISForGather ? retraceVarTalbot : retraceVar;

                        var["prevReservoirs"] = pPrevSuffixReservoirs;
                        var["suffixReuseRoundId"] = -1;
                        var["integrationPrefixId"] = integrationPrefixId;

                        int multiplier = mOptions.subpathSetting.useTalbotMISForGather ? mOptions.subpathSetting.finalGatherSuffixCount + 1 : 2;

                        if (mOptions.retraceScheduleType == ConditionalReSTIR::RetraceScheduleType::Naive)
                            pSuffixRetrace->execute(pRenderContext, mFrameDim.x, mFrameDim.y, 1);
                        else
                            pSuffixRetrace->execute(pRenderContext, multiplier * mOptions.subpathSetting.finalGatherSuffixCount * mFrameDim.x * mFrameDim.y, 1, 1);
                    }

                    {
                        FALCOR_PROFILE("FinalGatherIntegration");

                        prefixVar["reservoirs"] = mpReservoirs;
                        prefixVar["prevReservoirs"] = pPrevSuffixReservoirs;
                        prefixVar["suffixReuseRoundId"] = -1;
                        prefixVar["prefixReservoirs"] = mpPrefixReservoirs;
                        prefixVar["curPrefixLength"] = numLevels - iter;
                        prefixVar["integrationPrefixId"] = integrationPrefixId;
                        prefixVar["hasCanonicalSuffix"] = hasCanonicalSuffix;

                        mpSuffixResampling->execute(pRenderContext, mFrameDim.x, mFrameDim.y, 1);
                    }
                }
            }
        }

        mResetTemporalReservoirs = false;

        // prepare temporal data
        if (!mpScene->freeze)
        {
            if (mpTemporalVBuffer)
                pRenderContext->copyResource(mpTemporalVBuffer.get(), pVBuffer.get());
            mPrevCameraU = mpScene->getCamera()->getData().cameraU;
            mPrevCameraV = mpScene->getCamera()->getData().cameraV;
            mPrevCameraW = mpScene->getCamera()->getData().cameraW;
            mPrevJitterX = mpScene->getCamera()->getData().jitterX;
            mPrevJitterY = mpScene->getCamera()->getData().jitterY;
        }
    }
```

```cpp
    ShaderVar ConditionalReSTIRPass::bindSuffixResamplingVars(RenderContext* pRenderContext,
        ComputePass::SharedPtr pPass, std::string cbName, const Texture::SharedPtr& pVBuffer, const Texture::SharedPtr& pMotionVectors, bool bindPathTracer, bool bindVBuffer)
    {
        pPass["gScene"] = mpScene->getParameterBlock();
        auto rootVar = pPass->getRootVar();

        mpScene->setRaytracingShaderData(pRenderContext, rootVar);
        mpPixelDebug->prepareProgram(pPass->getProgram(), rootVar);
        mpPixelStats->prepareProgram(pPass->getProgram(), rootVar);

        auto var = rootVar["CB"][cbName];

        var["neighborOffsets"] = mpNeighborOffsets;

        var["motionVectors"] = pMotionVectors;

        var["params"].setBlob(mPathTracerParams);
        setShaderData(var["restir"]);

        // Bind the path tracer.
        if (bindPathTracer)
        {
            rootVar["gPathTracer"] = mpPathTracerBlock;
        }

        var["reservoirs"] = mpReservoirs;
        var["prevReservoirs"] = !mpScene->freeze ? mpPrevSuffixReservoirs : mpTempReservoirs; // doesn't matter, this will be binded differently for different passes

        var["prefixGBuffer"] = mpPrefixGBuffer;
        var["prevPrefixGBuffer"] = mpPrevPrefixGBuffer;

        var["neighborValidMask"] = mpNeighborValidMaskBuffer;

        if (bindVBuffer)
        {
            var["vbuffer"] = pVBuffer;
            var["temporalVbuffer"] = mpTemporalVBuffer;
        }
        return var;
    }
```
- 這段代碼是 ConditionalReSTIRPass 類中的 bindSuffixResamplingVars 函數。

- 這個函數用於綁定後綴重新採樣所需的變數和資源。以下是函數中的一些主要步驟：

    - 將場景相關的參數塞入後綴重新採樣的計算通道（Compute Pass）中。
    - 設置光線追蹤着色器數據。
    - 準備像素調試和統計程序。
    - 綁定後綴重新採樣計算通道的常量緩衝區。
    - 如果需要，綁定光線追蹤器。
    - 設置後綴重新採樣所需的變數，包括沖洗區域、前一個沖洗區域、前綴 GBuffer 等。
    - 如果需要，綁定 VBuffer（視圖緩衝區）和暫時 VBuffer。

- 總的來說，這個函數是用於準備後綴重新採樣所需的變數和資源，以便後續的計算通道能夠正確執行後綴重新採樣操作。

```cpp
    ShaderVar ConditionalReSTIRPass::bindSuffixResamplingOneVars(RenderContext* pRenderContext,
        ComputePass::SharedPtr pPass, std::string cbName, const Texture::SharedPtr& pMotionVectors, bool bindPathTracer)
    {
        pPass["gScene"] = mpScene->getParameterBlock();
        auto rootVar = pPass->getRootVar();

        mpScene->setRaytracingShaderData(pRenderContext, rootVar);
        mpPixelDebug->prepareProgram(pPass->getProgram(), rootVar);
        mpPixelStats->prepareProgram(pPass->getProgram(), rootVar);

        auto var = rootVar["CB"][cbName];

        var["motionVectors"] = pMotionVectors;

        var["params"].setBlob(mPathTracerParams);
        setShaderData(var["restir"]);

        // Bind the path tracer.
        if (bindPathTracer)
        {
            rootVar["gPathTracer"] = mpPathTracerBlock;
        }

        var["reservoirs"] = mpReservoirs;
        var["prevReservoirs"] = mpPrevSuffixReservoirs;

        var["prefixGBuffer"] = mpPrefixGBuffer;

        return var;
    }
```
- 這段代碼是 ConditionalReSTIRPass 類中的 bindSuffixResamplingOneVars 函數。

- 這個函數類似於前面提到的 bindSuffixResamplingVars 函數，不過它少了一些綁定的變數和資源。

- 具體來說，這個函數執行以下操作：

    - 將場景相關的參數塞入後綴重新採樣的計算通道（Compute Pass）中。
    - 設置光線追蹤着色器數據。
    - 準備像素調試和統計程序。
    - 綁定後綴重新採樣計算通道的常量緩衝區。
    - 如果需要，綁定光線追蹤器。
    - 設置後綴重新採樣所需的變數，包括沖洗區域、前一個沖洗區域、前綴 GBuffer 等。

- 總的來說，這個函數是用於準備後綴重新採樣所需的變數和資源，但與 bindSuffixResamplingVars 函數相比，它少了綁定 VBuffer（視圖緩衝區）和暫時 VBuffer 的步驟。

```cpp
    ShaderVar ConditionalReSTIRPass::bindPrefixResamplingVars(RenderContext* pRenderContext, ComputePass::SharedPtr pPass, std::string cbName, const Texture::SharedPtr& pVBuffer, const Texture::SharedPtr& pMotionVectors, bool bindPathTracer)
    {
        pPass["gScene"] = mpScene->getParameterBlock();
        auto rootVar = pPass->getRootVar();

        mpScene->setRaytracingShaderData(pRenderContext, rootVar);
        mpPixelDebug->prepareProgram(pPass->getProgram(), rootVar);
        mpPixelStats->prepareProgram(pPass->getProgram(), rootVar);

        auto var = rootVar["CB"][cbName];

        var["vbuffer"] = pVBuffer;
        var["temporalVbuffer"] = mpTemporalVBuffer;

        var["motionVectors"] = pMotionVectors;

        var["prevCameraU"] = mPrevCameraU;
        var["prevCameraV"] = mPrevCameraV;
        var["prevCameraW"] = mPrevCameraW;
        var["prevJitterX"] = mPrevJitterX;
        var["prevJitterY"] = mPrevJitterY;

        var["params"].setBlob(mPathTracerParams);
        setShaderData(var["restir"]);

        var["neighborValidMask"] = mpNeighborValidMaskBuffer;

        // Bind the path tracer.
        if (bindPathTracer)
        {
            rootVar["gPathTracer"] = mpPathTracerBlock;
        }

        return var;
    }
```
- 這段代碼是 ConditionalReSTIRPass 類中的 bindPrefixResamplingVars 函數。

- 這個函數的作用是綁定前綴重新採樣所需的變數和資源。

- 具體來說，這個函數執行以下操作：

    - 將場景相關的參數塞入前綴重新採樣的計算通道（Compute Pass）中。
    - 設置光線追蹤着色器數據。
    - 準備像素調試和統計程序。
    - 綁定前綴重新採樣計算通道的常量緩衝區。
    - 如果需要，綁定光線追蹤器。
    - 設置前綴重新採樣所需的變數，包括視圖緩衝區、暫時 VBuffer、運動向量、前一次的相機向量和抖動等。

- 總的來說，這個函數是用於準備前綴重新採樣所需的變數和資源。

```cpp
    Texture::SharedPtr ConditionalReSTIRPass::createNeighborOffsetTexture(uint32_t sampleCount)
    {
        std::unique_ptr<int8_t[]> offsets(new int8_t[sampleCount * 2]);
        const int R = 254;
        const float phi2 = 1.f / 1.3247179572447f;
        float u = 0.5f;
        float v = 0.5f;
        for (uint32_t index = 0; index < sampleCount * 2;)
        {
            u += phi2;
            v += phi2 * phi2;
            if (u >= 1.f) u -= 1.f;
            if (v >= 1.f) v -= 1.f;

            float rSq = (u - 0.5f) * (u - 0.5f) + (v - 0.5f) * (v - 0.5f);
            if (rSq > 0.25f) continue;

            offsets[index++] = int8_t((u - 0.5f) * R);
            offsets[index++] = int8_t((v - 0.5f) * R);
        }

        return Texture::create1D(sampleCount, ResourceFormat::RG8Snorm, 1, 1, offsets.get());
    }
```
- 這段代碼是 ConditionalReSTIRPass 類中的 createNeighborOffsetTexture 函數。

- 這個函數的作用是創建鄰域偏移紋理，用於在 Conditional ReSTIR 算法中存儲鄰域偏移信息。

- 具體來說，這個函數執行以下操作：

    - 通過輸入的樣本數創建一個一維紋理。
    - 使用黃金分割率計算鄰域偏移。
    - 將計算出的鄰域偏移填充到紋理中。

- 總的來說，這個函數是用於創建存儲鄰域偏移信息的紋理。

```cpp
    void ConditionalReSTIRPass::scriptBindings(pybind11::module& m)
    {
        // This library can be used from multiple render passes. If already registered, return immediately.
        if (pybind11::hasattr(m, "ConditionalReSTIROptions")) return;

        ScriptBindings::SerializableStruct<ConditionalReSTIR::SubpathReuseSettings> subpathsettings(m, "SubpathSettings");
#define field(f_) field(#f_, &ConditionalReSTIR::SubpathReuseSettings::f_)
        subpathsettings.field(useMMIS);

        subpathsettings.field(suffixSpatialNeighborCount);
        subpathsettings.field(suffixSpatialReuseRadius);
        subpathsettings.field(suffixSpatialReuseRounds);
        subpathsettings.field(suffixTemporalReuse);
        subpathsettings.field(temporalHistoryLength);
        // TODO: add fields
        subpathsettings.field(finalGatherSuffixCount);
        subpathsettings.field(prefixNeighborSearchRadius);
        subpathsettings.field(prefixNeighborSearchNeighborCount);

        subpathsettings.field(numIntegrationPrefixes);
        subpathsettings.field(generateCanonicalSuffixForEachPrefix);

#undef field


        ScriptBindings::SerializableStruct<Options> options(m, "ConditionalReSTIROptions");
#define field(f_) field(#f_, &Options::f_)

        options.field(subpathSetting);
        options.field(shiftMappingSettings);

#undef field

        ScriptBindings::SerializableStruct<ConditionalReSTIR::ShiftMappingSettings> shiftmappingsettings(m, "ShiftMappingSettings");
#define field(f_) field(#f_, &ConditionalReSTIR::ShiftMappingSettings::f_)
        shiftmappingsettings.field(localStrategyType);
        shiftmappingsettings.field(specularRoughnessThreshold);
        shiftmappingsettings.field(nearFieldDistanceThreshold);
#undef field
    }
}
```
- 這段代碼是 ConditionalReSTIRPass 類中的 scriptBindings 函數，它負責將 Conditional ReSTIR 相關的類型和設置綁定到 Python 中，以便在 Python 中使用這些類型和設置。

- 具體來說，這個函數執行以下操作：

    - 使用 pybind11 庫將 ConditionalReSTIR::SubpathReuseSettings、Options 和 ConditionalReSTIR::ShiftMappingSettings 這三個類型綁定到 Python 中。
    - 定義了這些類型的成員變量，並將它們綁定到 Python 中的相應成員。
    - 將這些類型綁定到 Python 中後，可以在 Python 中創建、設置和使用這些類型的實例，以及訪問它們的成員變量。

- 總的來說，這個函數的作用是實現了 Python 與 Conditional ReSTIR 相關類型和設置之間的互操作性。
