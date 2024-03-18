```cpp
#include "PathTracer.h"
#include "RenderGraph/RenderPassLibrary.h"
#include "RenderGraph/RenderPassHelpers.h"
#include "RenderGraph/RenderPassStandardFlags.h"
#include "Rendering/Lights/EmissiveUniformSampler.h"
```

```cpp
const RenderPass::Info PathTracer::kInfo { "PathTracer", "Reference path tracer." };
```
- 這段程式碼是在定義一個靜態成員變數 kInfo，該變數是類型為 RenderPass::Info 的常量，屬於 PathTracer 類別。它包含了兩個成員，分別是一個字串 "PathTracer" 和另一個字串 "Reference path tracer."。
- 通常情況下，類別中的靜態成員變數可以用來存儲與該類別相關的信息或設定，並且可以在整個程式中使用。在這裡，kInfo 可能是用來描述 PathTracer 這個渲染通道的基本信息，包括名稱和描述。

```cpp
namespace
{
    const std::string kGeneratePathsFilename = "RenderPasses/PathTracer/GeneratePaths.cs.slang";
    const std::string kTracePassFilename = "RenderPasses/PathTracer/TracePass.cs.slang";
    const std::string kResolvePassFilename = "RenderPasses/PathTracer/ResolvePass.cs.slang";
    const std::string kReflectTypesFile = "RenderPasses/PathTracer/ReflectTypes.cs.slang";

    const std::string kShaderModel = "6_5";

    // Render pass inputs and outputs.
    const std::string kInputVBuffer = "vbuffer";
    const std::string kInputMotionVectors = "mvec";
    const std::string kInputViewDir = "viewW";
    const std::string kInputSampleCount = "sampleCount";
```
- 這段程式碼定義了一些常量，這些常量通常用於指定檔案路徑、Shader Model 和渲染通道的輸入輸出。下面是每個常量的說明：

    - kGeneratePathsFilename：生成路徑的腳本檔案的路徑。
    - kTracePassFilename：追蹤通道的腳本檔案的路徑。
    - kResolvePassFilename：解析通道的腳本檔案的路徑。
    - kReflectTypesFile：反射類型的腳本檔案的路徑。
    - kShaderModel：使用的Shader Model 的版本。
    - kInputVBuffer：渲染通道的輸入緩衝區的名稱。
    - kInputMotionVectors：渲染通道的運動向量的輸入名稱。
    - kInputViewDir：渲染通道的視圖方向的輸入名稱。
    - kInputSampleCount：渲染通道的輸入樣本計數的名稱。

- 這些常量的使用可以幫助程式碼更容易地理解和維護，因為它們將重要的資訊集中在一個地方，並且可以在整個程式中重複使用。

```cpp
    const Falcor::ChannelList kInputChannels =
    {
        { kInputVBuffer,        "gVBuffer",         "Visibility buffer in packed format" },
        { kInputMotionVectors,  "gMotionVectors",   "Motion vector buffer (float format)", true /* optional */ },
        { kInputViewDir,        "gViewW",           "World-space view direction (xyz float format)", true /* optional */ },
        { kInputSampleCount,    "gSampleCount",     "Sample count buffer (integer format)", true /* optional */, ResourceFormat::R8Uint },
    };
```
- 這段程式碼定義了一個 ChannelList 常量 kInputChannels，它包含了一個渲染通道的輸入通道列表。每個通道都由以下資訊組成：

    - 通道名稱 (kInputVBuffer, kInputMotionVectors, kInputViewDir, kInputSampleCount)：這是通道的唯一識別符號。
    - Shader 中對應的變數名稱 ("gVBuffer", "gMotionVectors", "gViewW", "gSampleCount")：這是在 Shader 中使用的變數名稱，通常用於在 GPU 上訪問相應的資源。
    - 通道的描述 ("Visibility buffer in packed format", "Motion vector buffer (float format)", "World-space view direction (xyz float format)", "Sample count buffer (integer format)")：這是對通道功能或內容的說明，幫助程式開發人員了解該通道的作用。
    - 是否是可選的 (true 或 false)：這指示了該通道是否是可選的。如果是可選的，則可能在某些情況下可以省略，而不會導致渲染失敗。
    - 資源格式 (ResourceFormat::R8Uint)：這是通道資源的格式，通常用於指示通道中數據的類型和布局。

- ChannelList 通常用於定義渲染通道的輸入和輸出，以便在渲染過程中設置和訪問相關的資源。

```cpp
    const std::string kOutputColor = "color";
    const std::string kOutputSubColor = "subColor";
    const std::string kOutputVariance = "variance";
    const std::string kOutputAlbedo = "albedo";
    const std::string kOutputSpecularAlbedo = "specularAlbedo";
    const std::string kOutputIndirectAlbedo = "indirectAlbedo";
    const std::string kOutputNormal = "normal";
    const std::string kOutputReflectionPosW = "reflectionPosW";
    const std::string kOutputRayCount = "rayCount";
    const std::string kOutputPathLength = "pathLength";
    const std::string kOutputNRDDiffuseRadianceHitDist = "nrdDiffuseRadianceHitDist";
    const std::string kOutputNRDSpecularRadianceHitDist = "nrdSpecularRadianceHitDist";
    const std::string kOutputNRDEmission = "nrdEmission";
    const std::string kOutputNRDDiffuseReflectance = "nrdDiffuseReflectance";
    const std::string kOutputNRDSpecularReflectance = "nrdSpecularReflectance";
    const std::string kOutputNRDDeltaReflectionRadianceHitDist = "nrdDeltaReflectionRadianceHitDist";
    const std::string kOutputNRDDeltaReflectionReflectance = "nrdDeltaReflectionReflectance";
    const std::string kOutputNRDDeltaReflectionEmission = "nrdDeltaReflectionEmission";
    const std::string kOutputNRDDeltaReflectionNormWRoughMaterialID = "nrdDeltaReflectionNormWRoughMaterialID";
    const std::string kOutputNRDDeltaReflectionPathLength = "nrdDeltaReflectionPathLength";
    const std::string kOutputNRDDeltaReflectionHitDist = "nrdDeltaReflectionHitDist";
    const std::string kOutputNRDDeltaTransmissionRadianceHitDist = "nrdDeltaTransmissionRadianceHitDist";
    const std::string kOutputNRDDeltaTransmissionReflectance = "nrdDeltaTransmissionReflectance";
    const std::string kOutputNRDDeltaTransmissionEmission = "nrdDeltaTransmissionEmission";
    const std::string kOutputNRDDeltaTransmissionNormWRoughMaterialID = "nrdDeltaTransmissionNormWRoughMaterialID";
    const std::string kOutputNRDDeltaTransmissionPathLength = "nrdDeltaTransmissionPathLength";
    const std::string kOutputNRDDeltaTransmissionPosW = "nrdDeltaTransmissionPosW";
    const std::string kOutputNRDResidualRadianceHitDist = "nrdResidualRadianceHitDist";
```
- 這些都是代表渲染通道的輸出的常量，它們用於標識渲染通道生成的不同資訊或渲染結果。以下是每個常量的說明：

    - kOutputColor：渲染結果的顏色。
    - kOutputSubColor：額外渲染結果的顏色。
    - kOutputVariance：渲染結果的變異性。
    - kOutputAlbedo：表面的漫反射顏色。
    - kOutputSpecularAlbedo：表面的鏡面反射顏色。
    - kOutputIndirectAlbedo：表面的間接漫反射顏色。
    - kOutputNormal：表面的法向量。
    - kOutputReflectionPosW：反射點的世界空間位置。
    - kOutputRayCount：光線數量。
    - kOutputPathLength：光線的路徑長度。
    - kOutputNRDDiffuseRadianceHitDist：NRD漫反射輻射的命中距離。
    - kOutputNRDSpecularRadianceHitDist：NRD鏡面輻射的命中距離。
    - kOutputNRDEmission：NRD發射光。
    - kOutputNRDDiffuseReflectance：NRD漫反射反射率。
    - kOutputNRDSpecularReflectance：NRD鏡面反射反射率。
    - kOutputNRDDeltaReflectionRadianceHitDist：NRD Delta反射輻射的命中距離。
    - kOutputNRDDeltaReflectionReflectance：NRD Delta反射反射率。
    - kOutputNRDDeltaReflectionEmission：NRD Delta反射發射光。
    - kOutputNRDDeltaReflectionNormWRoughMaterialID：NRD Delta反射法向量、粗糙度和材質ID。
    - kOutputNRDDeltaReflectionPathLength：NRD Delta反射路徑長度。
    - kOutputNRDDeltaReflectionHitDist：NRD Delta反射的命中距離。
    - kOutputNRDDeltaTransmissionRadianceHitDist：NRD Delta透射輻射的命中距離。
    - kOutputNRDDeltaTransmissionReflectance：NRD Delta透射反射率。
    - kOutputNRDDeltaTransmissionEmission：NRD Delta透射發射光。
    - kOutputNRDDeltaTransmissionNormWRoughMaterialID：NRD Delta透射法向量、粗糙度和材質ID。
    - kOutputNRDDeltaTransmissionPathLength：NRD Delta透射路徑長度。
    - kOutputNRDDeltaTransmissionPosW：NRD Delta透射的世界空間位置。
    - kOutputNRDResidualRadianceHitDist：NRD殘留輻射的命中距離。

- 這些常量用於標識渲染通道生成的不同類型的資訊，例如顏色、法向量、輻射等，以便在後續處理中使用。

```cpp
    const Falcor::ChannelList kOutputChannels =
    {
        { kOutputColor,                                     "",     "Output color (linear)", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputSubColor, "", "Output color (linear)", true /* optional */, ResourceFormat::RGBA32Float},
        { kOutputVariance, "", "Output variance (avg X^2, avg X, var estimate)", true /* optional */, ResourceFormat::RGBA32Float},
        { kOutputAlbedo,                                    "",     "Output albedo (linear)", true /* optional */, ResourceFormat::RGBA8Unorm },
        { kOutputSpecularAlbedo,                            "",     "Output specular albedo (linear)", true /* optional */, ResourceFormat::RGBA8Unorm },
        { kOutputIndirectAlbedo,                            "",     "Output indirect albedo (linear)", true /* optional */, ResourceFormat::RGBA8Unorm },
        { kOutputNormal,                                    "",     "Output normal (linear)", true /* optional */, ResourceFormat::RGBA16Float },
        { kOutputReflectionPosW,                            "",     "Output reflection pos (world space)", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputRayCount,                                  "",     "Per-pixel ray count", true /* optional */, ResourceFormat::R32Uint },
        { kOutputPathLength,                                "",     "Per-pixel path length", true /* optional */, ResourceFormat::R32Uint },
        // NRD outputs
        { kOutputNRDDiffuseRadianceHitDist,                 "",     "Output demodulated diffuse color (linear) and hit distance", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputNRDSpecularRadianceHitDist,                "",     "Output demodulated specular color (linear) and hit distance", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputNRDEmission,                               "",     "Output primary surface emission", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputNRDDiffuseReflectance,                     "",     "Output primary surface diffuse reflectance", true /* optional */, ResourceFormat::RGBA16Float },
        { kOutputNRDSpecularReflectance,                    "",     "Output primary surface specular reflectance", true /* optional */, ResourceFormat::RGBA16Float },
        { kOutputNRDDeltaReflectionRadianceHitDist,         "",     "Output demodulated delta reflection color (linear)", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputNRDDeltaReflectionReflectance,             "",     "Output delta reflection reflectance color (linear)", true /* optional */, ResourceFormat::RGBA16Float },
        { kOutputNRDDeltaReflectionEmission,                "",     "Output delta reflection emission color (linear)", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputNRDDeltaReflectionNormWRoughMaterialID,    "",     "Output delta reflection world normal, roughness, and material ID", true /* optional */, ResourceFormat::RGB10A2Unorm },
        { kOutputNRDDeltaReflectionPathLength,              "",     "Output delta reflection path length", true /* optional */, ResourceFormat::R16Float },
        { kOutputNRDDeltaReflectionHitDist,                 "",     "Output delta reflection hit distance", true /* optional */, ResourceFormat::R16Float },
        { kOutputNRDDeltaTransmissionRadianceHitDist,       "",     "Output demodulated delta transmission color (linear)", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputNRDDeltaTransmissionReflectance,           "",     "Output delta transmission reflectance color (linear)", true /* optional */, ResourceFormat::RGBA16Float },
        { kOutputNRDDeltaTransmissionEmission,              "",     "Output delta transmission emission color (linear)", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputNRDDeltaTransmissionNormWRoughMaterialID,  "",     "Output delta transmission world normal, roughness, and material ID", true /* optional */, ResourceFormat::RGB10A2Unorm },
        { kOutputNRDDeltaTransmissionPathLength,            "",     "Output delta transmission path length", true /* optional */, ResourceFormat::R16Float },
        { kOutputNRDDeltaTransmissionPosW,                  "",     "Output delta transmission position", true /* optional */, ResourceFormat::RGBA32Float },
        { kOutputNRDResidualRadianceHitDist,                "",     "Output residual color (linear) and hit distance", true /* optional */, ResourceFormat::RGBA32Float },
    };
```
- 這是用於描述渲染通道輸出的通道列表，其中包含了不同類型資訊的輸出通道以及它們的屬性。以下是每個通道的說明：

    - kOutputColor：線性的輸出顏色。
    - kOutputSubColor：額外的線性輸出顏色。
    - kOutputVariance：平均X^2、平均X和變異度估計的輸出變異度。
    - kOutputAlbedo：線性的表面漫反射顏色。
    - kOutputSpecularAlbedo：線性的表面鏡面反射顏色。
    - kOutputIndirectAlbedo：線性的表面間接漫反射顏色。
    - kOutputNormal：線性的表面法向量。
    - kOutputReflectionPosW：世界空間中的反射位置。
    - kOutputRayCount：每個像素的光線數量。
    - kOutputPathLength：每個像素的光線路徑長度。
    - kOutputNRDDiffuseRadianceHitDist：解調漫反射輻射的命中距離。
    - kOutputNRDSpecularRadianceHitDist：解調鏡面輻射的命中距離。
    - kOutputNRDEmission：主表面發射光。
    - kOutputNRDDiffuseReflectance：主表面漫反射反射率。
    - kOutputNRDSpecularReflectance：主表面鏡面反射反射率。
    - kOutputNRDDeltaReflectionRadianceHitDist：解調Delta反射輻射的命中距離。
    - kOutputNRDDeltaReflectionReflectance：Delta反射反射率的顏色。
    - kOutputNRDDeltaReflectionEmission：Delta反射發射光的顏色。
    - kOutputNRDDeltaReflectionNormWRoughMaterialID：Delta反射的世界法向量、粗糙度和材質ID。
    - kOutputNRDDeltaReflectionPathLength：Delta反射的路徑長度。
    - kOutputNRDDeltaReflectionHitDist：Delta反射的命中距離。
    - kOutputNRDDeltaTransmissionRadianceHitDist：解調Delta透射輻射的命中距離。
    - kOutputNRDDeltaTransmissionReflectance：Delta透射反射率的顏色。
    - kOutputNRDDeltaTransmissionEmission：Delta透射發射光的顏色。
    - kOutputNRDDeltaTransmissionNormWRoughMaterialID：Delta透射的世界法向量、粗糙度和材質ID。
    - kOutputNRDDeltaTransmissionPathLength：Delta透射的路徑長度。
    - kOutputNRDDeltaTransmissionPosW：Delta透射的位置。
    - kOutputNRDResidualRadianceHitDist：殘餘顏色和命中距離。

- 這些通道描述了渲染過程中生成的各種資訊，並且可以在後續處理過程中使用。

```cpp
    // UI variables.
    const Gui::DropdownList kColorFormatList =
    {
        { (uint32_t)ColorFormat::RGBA32F, "RGBA32F (128bpp)" },
        { (uint32_t)ColorFormat::LogLuvHDR, "LogLuvHDR (32bpp)" },
    };

    const Gui::DropdownList kMISHeuristicList =
    {
        { (uint32_t)MISHeuristic::Balance, "Balance heuristic" },
        { (uint32_t)MISHeuristic::PowerTwo, "Power heuristic (exp=2)" },
        { (uint32_t)MISHeuristic::PowerExp, "Power heuristic" },
    };

    const Gui::DropdownList kEmissiveSamplerList =
    {
        { (uint32_t)EmissiveLightSamplerType::Uniform, "Uniform" },
        { (uint32_t)EmissiveLightSamplerType::LightBVH, "LightBVH" },
        { (uint32_t)EmissiveLightSamplerType::Power, "Power" },
    };

    const Gui::DropdownList kLODModeList =
    {
        { (uint32_t)TexLODMode::Mip0, "Mip0" },
        { (uint32_t)TexLODMode::RayDiffs, "Ray Diffs" }
    };

    const Gui::DropdownList kRenderModePreset = {
        {0, "CRIS/Conditional ReSTIR"},
        {1, "MMIS"},
        {2, "Path Tracing"}
    };

    const Gui::DropdownList kDIMode = {
        {0, "Enable DI"},
        {1, "Disable DI"},
        {2, "Disable DI through specular chain"},
    };
```
- 這是用於用戶界面(UI)的變數，其中包含了下拉列表的選項。以下是每個列表的說明：

    - kColorFormatList：顏色格式的下拉列表，包括RGBA32F和LogLuvHDR。
    - kMISHeuristicList：多重重要性取樣(MIS)啟發式的下拉列表，包括平衡啟發式、指數為2的幂次啟發式和指數幂次啟發式。
    - kEmissiveSamplerList：發射光取樣器類型的下拉列表，包括均勻取樣、光線包圍盒(LightBVH)取樣和功率取樣。
    - kLODModeList：紋理級別詳細度(LOD)模式的下拉列表，包括Mip0和Ray Diffs。
    - kRenderModePreset：渲染模式預設值的下拉列表，包括CRIS/Conditional ReSTIR、MMIS和Path Tracing。
    - kDIMode：直接照明(DI)模式的下拉列表，包括啟用DI、禁用DI和通過鏡面鏈禁用DI。

- 這些下拉列表提供了用於交互式設置和選擇不同選項的界面元素。

```cpp
    // Scripting options.
    const std::string kSamplesPerPixel = "samplesPerPixel";
    const std::string kTotalSamplesPerPixel = "totalSamplesPerPixel";
    const std::string kMaxSurfaceBounces = "maxSurfaceBounces";
    const std::string kMaxDiffuseBounces = "maxDiffuseBounces";
    const std::string kMaxSpecularBounces = "maxSpecularBounces";
    const std::string kMaxTransmissionBounces = "maxTransmissionBounces";

    const std::string kSampleGenerator = "sampleGenerator";
    const std::string kFixedSeed = "fixedSeed";
    const std::string kUseBSDFSampling = "useBSDFSampling";
    const std::string kUseRussianRoulette = "useRussianRoulette";
    const std::string kUseLambertianDiffuse = "useLambertianDiffuse";
    const std::string kDisableDirectIllumination = "disableDirectIllumination";
    const std::string kDisableGeneralizedDirectIllumination = "disableGeneralizedDirectIllumination";
    const std::string kDisableDiffuse = "disableDiffuse";
    const std::string kDisableSpecular = "disableSpecular";
    const std::string kDisableTranslucency = "disableTranslucency";

    const std::string kUseNEE = "useNEE";
    const std::string kUseMIS = "useMIS";
    const std::string kMISHeuristic = "misHeuristic";
    const std::string kMISPowerExponent = "misPowerExponent";
    const std::string kEmissiveSampler = "emissiveSampler";
    const std::string kLightBVHOptions = "lightBVHOptions";
    const std::string kUseRTXDI = "useRTXDI";
    const std::string kRTXDIOptions = "RTXDIOptions";
    const std::string kUseReSTIR = "useConditionalReSTIR";
    const std::string kConditionalReSTIROptions = "ConditionalReSTIROptions";

    const std::string kUseAlphaTest = "useAlphaTest";
    const std::string kAdjustShadingNormals = "adjustShadingNormals";
    const std::string kMaxNestedMaterials = "maxNestedMaterials";
    const std::string kUseLightsInDielectricVolumes = "useLightsInDielectricVolumes";
    const std::string kDisableCaustics = "disableCaustics";
    const std::string kSpecularRoughnessThreshold = "specularRoughnessThreshold";
    const std::string kPrimaryLodMode = "primaryLodMode";
    const std::string kLODBias = "lodBias";

    const std::string kOutputSize = "outputSize";
    const std::string kFixedOutputSize = "fixedOutputSize";
    const std::string kColorFormat = "colorFormat";
    const std::string kSeedOffset = "seedOffset";

    const std::string kUseNRDDemodulation = "useNRDDemodulation";
}
```
- 這些是用於腳本選項的常量字符串，用於指定各種渲染和效果參數。以下是每個常量的說明：

    - kSamplesPerPixel：每像素的採樣數。
    - kTotalSamplesPerPixel：總採樣數。
    - kMaxSurfaceBounces：表面反射的最大反彈次數。
    - kMaxDiffuseBounces：漫射反射的最大反彈次數。
    - kMaxSpecularBounces：鏡面反射的最大反彈次數。
    - kMaxTransmissionBounces：透射的最大反彈次數。
    - kSampleGenerator：採樣器的類型。
    - kFixedSeed：固定種子。
    - kUseBSDFSampling：是否使用BSDF採樣。
    - kUseRussianRoulette：是否使用俄羅斯輪盤法。
    - kUseLambertianDiffuse：是否使用Lambertian漫反射。
    - kDisableDirectIllumination：是否禁用直接照明。
    - kDisableGeneralizedDirectIllumination：是否禁用通用直接照明。
    - kDisableDiffuse：是否禁用漫射。
    - kDisableSpecular：是否禁用鏡面反射。
    - kDisableTranslucency：是否禁用半透明。
    - kUseNEE：是否使用下一樣本估計（Next Event Estimation）。
    - kUseMIS：是否使用多重重要性採樣（Multiple Importance Sampling）。
    - kMISHeuristic：多重重要性採樣的啟發式方法。
    - kMISPowerExponent：多重重要性採樣中的指數。
    - kEmissiveSampler：發光光源的採樣器類型。
    - kLightBVHOptions：光源包圍盒層次結構的選項。
    - kUseRTXDI：是否使用RTXDI（Real-Time Ray Tracing Denoiser and Illuminator）。
    - kRTXDIOptions：RTXDI的選項。
    - kUseReSTIR：是否使用條件式ReSTIR（Conditional ReSTIR）。
    - kConditionalReSTIROptions：條件式ReSTIR的選項。
    - kUseAlphaTest：指定是否使用 alpha 測試。
    - kAdjustShadingNormals：指定是否調整表面法線。
    - kMaxNestedMaterials：指定最大嵌套材質數量。
    - kUseLightsInDielectricVolumes：指定是否在折射體積中使用燈光。
    - kDisableCaustics：指定是否禁用焦散。
    - kSpecularRoughnessThreshold：指定用於鏡面反射粗糙度的閾值。
    - kPrimaryLodMode：指定主要細節層級的模式。
    - kLODBias：指定細節層級的偏差。
    - kOutputSize：指定輸出大小。
    - kFixedOutputSize：指定固定的輸出大小。
    - kColorFormat：指定顏色格式。
    - kSeedOffset：指定種子偏移量。
    - kUseNRDDemodulation：指定是否使用 NRDDemodulation。

等等。這些常量用於指定腳本中各種參數和選項的名稱，用於控制渲染器的各種設置和選項，以實現所需的渲染效果。

```
extern "C" FALCOR_API_EXPORT void getPasses(Falcor::RenderPassLibrary& lib)
{
    lib.registerPass(PathTracer::kInfo, PathTracer::create);
    ScriptBindings::registerBinding(PathTracer::registerBindings);
    ScriptBindings::registerBinding(ConditionalReSTIRPass::scriptBindings);
}
```
- 調用 RenderPassLibrary 類別的 registerPass() 方法，用於註冊一個渲染通道（Render Pass）。PathTracer::kInfo 用於指定渲染通道的相關資訊，而 PathTracer::create 則是用於創建渲染通道實例的函數。

```
void PathTracer::registerBindings(pybind11::module& m)
{
    pybind11::enum_<ColorFormat> colorFormat(m, "ColorFormat");
    colorFormat.value("RGBA32F", ColorFormat::RGBA32F);
    colorFormat.value("LogLuvHDR", ColorFormat::LogLuvHDR);

    pybind11::enum_<MISHeuristic> misHeuristic(m, "MISHeuristic");
    misHeuristic.value("Balance", MISHeuristic::Balance);
    misHeuristic.value("PowerTwo", MISHeuristic::PowerTwo);
    misHeuristic.value("PowerExp", MISHeuristic::PowerExp);

    pybind11::class_<PathTracer, RenderPass, PathTracer::SharedPtr> pass(m, "PathTracer");
    pass.def_property_readonly("pixelStats", &PathTracer::getPixelStats);

    pass.def_property("useFixedSeed",
        [](const PathTracer* pt) { return pt->mParams.useFixedSeed ? true : false; },
        [](PathTracer* pt, bool value) { pt->mParams.useFixedSeed = value ? 1 : 0; }
    );
    pass.def_property("fixedSeed",
        [](const PathTracer* pt) { return pt->mParams.fixedSeed; },
        [](PathTracer* pt, uint32_t value) { pt->mParams.fixedSeed = value; }
    );
}
```
- 這段程式碼是使用C++和pybind11庫（用於將C++代碼綁定到Python中）撰寫的，目的是將PathTracer這個渲染通道的功能暴露給Python語言使用。

    - void PathTracer::registerBindings(pybind11::module& m)：這是一個名為registerBindings的成員函數，屬於PathTracer類。它接受一個pybind11::module對象的引用作為參數，這個對象代表了一個Python模塊，用於定義Python的模塊和類。
    - pybind11::enum_<ColorFormat> colorFormat(m, "ColorFormat")：這行程式碼創建了一個名為ColorFormat的枚舉型別，並將它與Python模塊m綁定。
    - pybind11::class_<PathTracer, RenderPass, PathTracer::SharedPtr> pass(m, "PathTracer")：這行程式碼創建了一個Python類型PathTracer，並將它與Python模塊m綁定。這個類型是從C++類型PathTracer繼承而來的，同時也是RenderPass類型的子類。PathTracer::SharedPtr可能是指向PathTracer的智能指標類型。
    - pass.def_property_readonly("pixelStats", &PathTracer::getPixelStats)：這行程式碼定義了一個只讀屬性pixelStats，它將C++中的PathTracer::getPixelStats方法暴露給Python。
    - pass.def_property("useFixedSeed", ...)和pass.def_property("fixedSeed", ...)：這兩行程式碼定義了兩個可讀可寫的屬性useFixedSeed和fixedSeed，分別將C++中的成員變量與Python屬性綁定起來，從而可以在Python中設置和訪問它們。

- 總的來說，這段程式碼的功能是將PathTracer這個渲染通道的部分功能暴露給Python，以便Python語言可以使用這些功能。

```
PathTracer::SharedPtr PathTracer::create(RenderContext* pRenderContext, const Dictionary& dict)
{
    return SharedPtr(new PathTracer(dict));
}
```
- 這段程式碼是用於創建 PathTracer 渲染通道的函數。

    - PathTracer::SharedPtr：這表示PathTracer類的智能指標類型，通常用於指向PathTracer類的對象。
    - PathTracer::create(RenderContext* pRenderContext, const Dictionary& dict)：這是一個靜態成員函數，用於創建PathTracer類的實例。它接受兩個參數，pRenderContext是指向渲染上下文的指標，而dict是一個字典，用於傳遞創建PathTracer實例所需的參數。
    - return SharedPtr(new PathTracer(dict));：這行程式碼創建了一個PathTracer的實例，並通過智能指標SharedPtr封裝起來，然後返回給調用方。在這個過程中，使用了dict參數來初始化PathTracer類的實例。

- 總的來說，這段程式碼的功能是定義了一個靜態成員函數，用於創建PathTracer渲染通道的實例。

```
PathTracer::PathTracer(const Dictionary& dict)
    : RenderPass(kInfo)
{
    if (!gpDevice->isShaderModelSupported(Device::ShaderModel::SM6_5))
    {
        throw RuntimeError("PathTracer: Shader Model 6.5 is not supported by the current device");
    }
    if (!gpDevice->isFeatureSupported(Device::SupportedFeatures::RaytracingTier1_1))
    {
        throw RuntimeError("PathTracer: Raytracing Tier 1.1 is not supported by the current device");
    }

    parseDictionary(dict);
    validateOptions();

    // Create sample generator.
    mpSampleGenerator = SampleGenerator::create(mStaticParams.sampleGenerator);

    // Create resolve pass. This doesn't depend on the scene so can be created here.
    auto defines = mStaticParams.getDefines(*this);
    mpResolvePass = ComputePass::create(Program::Desc(kResolvePassFilename).setShaderModel(kShaderModel).csEntry("main"), defines, false);

    // Note: The other programs are lazily created in updatePrograms() because a scene needs to be present when creating them.

    mpPixelStats = PixelStats::create();
    mpPixelDebug = PixelDebug::create();
}
```
- 這是一個名為 PathTracer 的類別的構造函式，它接受一個名為 dict 的字典作為參數，並初始化了一個路徑追踪器。以下是構造函式的主要步驟：

    1. 首先，它檢查當前設備是否支援所需的Shader Model和Raytracing功能。如果不支援，則會拋出異常。
    2. 然後，它解析並驗證字典中的選項。
    3. 接著，它創建了一個樣本生成器 (SampleGenerator)。
    4. 然後，它創建了一個解析通道 (Resolve Pass)。這個解析通道不依賴於場景，因此可以在此處創建。
    5. 最後，它創建了一些其他的像素統計 (PixelStats) 和像素調試 (PixelDebug) 實例。

- 總的來說，這個構造函式初始化了一個路徑追踪器，並準備好了所需的相關資源和程序，以便在後續的渲染過程中使用。

```cpp
void PathTracer::parseDictionary(const Dictionary& dict)
{
    for (const auto& [key, value] : dict)
    {
        // Rendering parameters
        if (key == kSamplesPerPixel) mParams.samplesPerPixel = value;
        else if (key == kMaxSurfaceBounces) mStaticParams.maxSurfaceBounces = value;
        else if (key == kMaxDiffuseBounces) mStaticParams.maxDiffuseBounces = value;
        else if (key == kMaxSpecularBounces) mStaticParams.maxSpecularBounces = value;
        else if (key == kMaxTransmissionBounces) mStaticParams.maxTransmissionBounces = value;

        // Sampling parameters
        else if (key == kSampleGenerator) mStaticParams.sampleGenerator = value;
        else if (key == kFixedSeed) { mParams.fixedSeed = value; mParams.useFixedSeed = true; }
        else if (key == kUseBSDFSampling) mStaticParams.useBSDFSampling = value;
        else if (key == kUseRussianRoulette) mStaticParams.useRussianRoulette = value;
        else if (key == kUseLambertianDiffuse) mStaticParams.useLambertianDiffuse = value;
        else if (key == kDisableDirectIllumination) mParams.DIMode = value ? 1 : 0;
        // don't use kDisableDirectIllumination and kDisableGeneralizedDirectIllumination in the same script
        else if (key == kDisableGeneralizedDirectIllumination) mParams.DIMode = value ? 2 : 0;
        else if (key == kDisableDiffuse) mStaticParams.disableDiffuse = value;
        else if (key == kDisableSpecular) mStaticParams.disableSpecular = value;
        else if (key == kDisableTranslucency) mStaticParams.disableTranslucency = value;
        else if (key == kUseNEE) mStaticParams.useNEE = value;
        else if (key == kUseMIS) mStaticParams.useMIS = value;
        else if (key == kMISHeuristic) mStaticParams.misHeuristic = value;
        else if (key == kMISPowerExponent) mStaticParams.misPowerExponent = value;
        else if (key == kEmissiveSampler) mStaticParams.emissiveSampler = value;
        else if (key == kLightBVHOptions) mLightBVHOptions = value;
        else if (key == kUseRTXDI) mStaticParams.useRTXDI = value;
        else if (key == kUseReSTIR) mParams.useConditionalReSTIR = value;
        else if (key == kRTXDIOptions) mRTXDIOptions = value;
        else if (key == kConditionalReSTIROptions) mConditionalReSTIROptions = value;

        // Material parameters
        else if (key == kUseAlphaTest) mStaticParams.useAlphaTest = value;
        else if (key == kAdjustShadingNormals) mStaticParams.adjustShadingNormals = value;
        else if (key == kMaxNestedMaterials) mStaticParams.maxNestedMaterials = value;
        else if (key == kUseLightsInDielectricVolumes) mStaticParams.useLightsInDielectricVolumes = value;
        else if (key == kDisableCaustics) mStaticParams.disableCaustics = value;
        else if (key == kSpecularRoughnessThreshold) mParams.specularRoughnessThreshold = value;
        else if (key == kPrimaryLodMode) mStaticParams.primaryLodMode = value;
        else if (key == kLODBias) mParams.lodBias = value;

        // Denoising parameters
        else if (key == kUseNRDDemodulation) mStaticParams.useNRDDemodulation = value;

        // Output parameters
        else if (key == kOutputSize) mOutputSizeSelection = value;
        else if (key == kFixedOutputSize) mFixedOutputSize = value;
        else if (key == kColorFormat) mStaticParams.colorFormat = value;
        else if (key == kSeedOffset) mSeedOffset = value;

        else logWarning("Unknown field '{}' in PathTracer dictionary.", key);
    }

    if (dict.keyExists(kMaxSurfaceBounces))
    {
        // Initialize bounce counts to 'maxSurfaceBounces' if they weren't explicitly set.
        if (!dict.keyExists(kMaxDiffuseBounces)) mStaticParams.maxDiffuseBounces = mStaticParams.maxSurfaceBounces;
        if (!dict.keyExists(kMaxSpecularBounces)) mStaticParams.maxSpecularBounces = mStaticParams.maxSurfaceBounces;
        if (!dict.keyExists(kMaxTransmissionBounces)) mStaticParams.maxTransmissionBounces = mStaticParams.maxSurfaceBounces;
    }
    else
    {
        // Initialize surface bounces.
        mStaticParams.maxSurfaceBounces = std::max(mStaticParams.maxDiffuseBounces, std::max(mStaticParams.maxSpecularBounces, mStaticParams.maxTransmissionBounces));
    }

    bool maxSurfaceBouncesNeedsAdjustment =
        mStaticParams.maxSurfaceBounces < mStaticParams.maxDiffuseBounces ||
        mStaticParams.maxSurfaceBounces < mStaticParams.maxSpecularBounces ||
        mStaticParams.maxSurfaceBounces < mStaticParams.maxTransmissionBounces;

    // Show a warning if maxSurfaceBounces will be adjusted in validateOptions().
    if (dict.keyExists(kMaxSurfaceBounces) && maxSurfaceBouncesNeedsAdjustment)
    {
        logWarning("'{}' is set lower than '{}', '{}' or '{}' and will be increased.", kMaxSurfaceBounces, kMaxDiffuseBounces, kMaxSpecularBounces, kMaxTransmissionBounces);
    }
}
```
- 這段程式碼是 PathTracer 類別中的 parseDictionary 函式，用於解析傳入的字典並設置相應的參數。這裡列出了一些主要的操作：

    1. 逐項檢查字典中的鍵值對，並根據鍵值設置相應的參數。例如，kSamplesPerPixel 對應到 mParams.samplesPerPixel，kMaxSurfaceBounces 對應到 mStaticParams.maxSurfaceBounces，以此類推。
    2. 如果字典中存在 kMaxSurfaceBounces 鍵，則初始化反射、漫反射和折射的最大反射次數，並確保它們不會超過表面最大反射次數。
    3. 如果表面最大反射次數需要調整，則顯示警告訊息。

- 總的來說，這個函式是用於從字典中提取參數並初始化路徑追踪器的各種設定。

```cpp
void PathTracer::validateOptions()
{
    if (mParams.specularRoughnessThreshold < 0.f || mParams.specularRoughnessThreshold > 1.f)
    {
        logWarning("'specularRoughnessThreshold' has invalid value. Clamping to range [0,1].");
        mParams.specularRoughnessThreshold = clamp(mParams.specularRoughnessThreshold, 0.f, 1.f);
    }

    // Static parameters.
    if (mParams.samplesPerPixel < 1 || mParams.samplesPerPixel > kMaxSamplesPerPixel)
    {
        logWarning("'samplesPerPixel' must be in the range [1, {}]. Clamping to this range.", kMaxSamplesPerPixel);
        mParams.samplesPerPixel = std::clamp(mParams.samplesPerPixel, 1, (int)kMaxSamplesPerPixel);
    }

    auto clampBounces = [] (uint32_t& bounces, const std::string& name)
    {
        if (bounces > kMaxBounces)
        {
            logWarning("'{}' exceeds the maximum supported bounces. Clamping to {}.", name, kMaxBounces);
            bounces = kMaxBounces;
        }
    };

    clampBounces(mStaticParams.maxSurfaceBounces, kMaxSurfaceBounces);
    clampBounces(mStaticParams.maxDiffuseBounces, kMaxDiffuseBounces);
    clampBounces(mStaticParams.maxSpecularBounces, kMaxSpecularBounces);
    clampBounces(mStaticParams.maxTransmissionBounces, kMaxTransmissionBounces);

    // Make sure maxSurfaceBounces is at least as many as any of diffuse, specular or transmission.
    uint32_t minSurfaceBounces = std::max(mStaticParams.maxDiffuseBounces, std::max(mStaticParams.maxSpecularBounces, mStaticParams.maxTransmissionBounces));
    mStaticParams.maxSurfaceBounces = std::max(mStaticParams.maxSurfaceBounces, minSurfaceBounces);

    if (mStaticParams.primaryLodMode == TexLODMode::RayCones)
    {
        logWarning("Unsupported tex lod mode. Defaulting to Mip0.");
        mStaticParams.primaryLodMode = TexLODMode::Mip0;
    }
}
```
- 這段程式碼是 PathTracer 類別中的 validateOptions 函式，用於驗證路徑追踪器的選項是否有效。這裡列出了一些主要的操作：

    1. 驗證 specularRoughnessThreshold 是否在合理的範圍內，如果不在範圍內則將其夾在 [0,1] 的範圍內。
    2. 驗證 samplesPerPixel 是否在合理的範圍內，如果不在範圍內則將其夾在 [1, kMaxSamplesPerPixel] 的範圍內。
    3. 如果超過了最大支援的反射次數（bounces），則將其夾在合理的範圍內。
    4. 確保表面最大反射次數（maxSurfaceBounces）至少等於漫反射、高光反射和折射中的最大次數。
    5. 如果選擇的 LOD 模式不受支援，則將其默認為 Mip0。

- 總的來說，這個函式是用於確保路徑追踪器的各種選項在合理的範圍內，並且如果有必要，對其進行修正以符合系統的限制或預期行為。

```cpp
Dictionary PathTracer::getScriptingDictionary()
{
    if (auto lightBVHSampler = std::dynamic_pointer_cast<LightBVHSampler>(mpEmissiveSampler))
    {
        mLightBVHOptions = lightBVHSampler->getOptions();
    }

    Dictionary d;

    // Rendering parameters
    d[kSamplesPerPixel] = mParams.samplesPerPixel;
    d[kMaxSurfaceBounces] = mStaticParams.maxSurfaceBounces;
    d[kMaxDiffuseBounces] = mStaticParams.maxDiffuseBounces;
    d[kMaxSpecularBounces] = mStaticParams.maxSpecularBounces;
    d[kMaxTransmissionBounces] = mStaticParams.maxTransmissionBounces;

    // Sampling parameters
    d[kSampleGenerator] = mStaticParams.sampleGenerator;
    if (mParams.useFixedSeed) d[kFixedSeed] = mParams.fixedSeed;
    d[kUseBSDFSampling] = mStaticParams.useBSDFSampling;
    d[kUseRussianRoulette] = mStaticParams.useRussianRoulette;
    d[kUseLambertianDiffuse] = mStaticParams.useLambertianDiffuse;
    d[kUseNEE] = mStaticParams.useNEE;
    d[kUseMIS] = mStaticParams.useMIS;
    d[kMISHeuristic] = mStaticParams.misHeuristic;
    d[kMISPowerExponent] = mStaticParams.misPowerExponent;
    d[kEmissiveSampler] = mStaticParams.emissiveSampler;
    if (mStaticParams.emissiveSampler == EmissiveLightSamplerType::LightBVH) d[kLightBVHOptions] = mLightBVHOptions;
    d[kUseRTXDI] = mStaticParams.useRTXDI;
    d[kRTXDIOptions] = mRTXDIOptions;
    d[kUseReSTIR] = mParams.useConditionalReSTIR;
    d[kConditionalReSTIROptions] = mConditionalReSTIROptions;

    // Material parameters
    d[kUseAlphaTest] = mStaticParams.useAlphaTest;
    d[kAdjustShadingNormals] = mStaticParams.adjustShadingNormals;
    d[kMaxNestedMaterials] = mStaticParams.maxNestedMaterials;
    d[kUseLightsInDielectricVolumes] = mStaticParams.useLightsInDielectricVolumes;
    d[kDisableCaustics] = mStaticParams.disableCaustics;
    d[kSpecularRoughnessThreshold] = mParams.specularRoughnessThreshold;
    d[kPrimaryLodMode] = mStaticParams.primaryLodMode;
    d[kLODBias] = mParams.lodBias;

    // Denoising parameters
    d[kUseNRDDemodulation] = mStaticParams.useNRDDemodulation;

    // Output parameters
    d[kOutputSize] = mOutputSizeSelection;
    if (mOutputSizeSelection == RenderPassHelpers::IOSize::Fixed) d[kFixedOutputSize] = mFixedOutputSize;
    d[kColorFormat] = mStaticParams.colorFormat;

    return d;
}
```
- 這段程式碼是 PathTracer 類別中的 getScriptingDictionary 函式，用於獲取路徑追踪器的腳本字典，將其用於配置和腳本化。這個函式將路徑追踪器的各種參數和選項轉換為一個字典，其中包含了以下項目：

    1. 渲染參數：如每像素採樣次數和最大反射次數等。
    2. 採樣參數：如採樣器類型、是否使用固定種子等。
    3. 材質參數：如是否使用 alpha 測試、最大嵌套材質數量等。
    4. 陰影處理參數：如是否啟用 NRDDemodulation（非線性陰影抑制）等。
    5. 輸出參數：如輸出圖像大小、顏色格式等。

- 函式會檢查是否使用了特定類型的發光體採樣器（LightBVHSampler），如果是，則會從該採樣器中獲取額外的選項。
- 最後，將所有參數和選項添加到字典中並返回。這樣做是為了使得這些參數可以在腳本中使用，以便對路徑追踪器進行配置和調整。

```cpp
RenderPassReflection PathTracer::reflect(const CompileData& compileData)
{
    RenderPassReflection reflector;
    const uint2 sz = RenderPassHelpers::calculateIOSize(mOutputSizeSelection, mFixedOutputSize, compileData.defaultTexDims);

    addRenderPassInputs(reflector, kInputChannels);
    addRenderPassOutputs(reflector, kOutputChannels, ResourceBindFlags::UnorderedAccess, sz);
    return reflector;
}
```
- 這段程式碼是 PathTracer 類別中的 reflect 函式，用於反射路徑追踪器的渲染通道。在圖形編譯數據（CompileData）的基礎上，它建立了一個 RenderPassReflection 對象，該對象描述了渲染通道的輸入和輸出。

- 具體來說：

    1. 創建了一個 RenderPassReflection 對象（reflector）。
    2. 使用 RenderPassHelpers::calculateIOSize 函式計算了輸出圖像的尺寸（sz），該函式基於輸出大小選擇（mOutputSizeSelection）、固定輸出尺寸（mFixedOutputSize）以及編譯數據的默認紋理尺寸。
    3. 使用 addRenderPassInputs 函式將輸入通道（kInputChannels）添加到反射器中。
    4. 使用 addRenderPassOutputs 函式將輸出通道（kOutputChannels）添加到反射器中，並指定了資源綁定標誌（ResourceBindFlags::UnorderedAccess）和輸出圖像的尺寸（sz）。
    5. 返回反射器（reflector）。

- 這個函式的作用是描述路徑追踪器的輸入和輸出，以便在後續的渲染管線中使用。

```
void PathTracer::setFrameDim(const uint2 frameDim)
{
    auto prevFrameDim = mParams.frameDim;
    auto prevScreenTiles = mParams.screenTiles;

    mParams.frameDim = frameDim;
    if (mParams.frameDim.x > kMaxFrameDimension || mParams.frameDim.y > kMaxFrameDimension)
    {
        throw RuntimeError("Frame dimensions up to {} pixels width/height are supported.", kMaxFrameDimension);
    }

    // Tile dimensions have to be powers-of-two.
    FALCOR_ASSERT(isPowerOf2(kScreenTileDim.x) && isPowerOf2(kScreenTileDim.y));
    FALCOR_ASSERT(kScreenTileDim.x == (1 << kScreenTileBits.x) && kScreenTileDim.y == (1 << kScreenTileBits.y));
    mParams.screenTiles = div_round_up(mParams.frameDim, kScreenTileDim);

    if (mParams.frameDim != prevFrameDim || mParams.screenTiles != prevScreenTiles)
    {
        mVarsChanged = true;
    }
}
```
- 這段程式碼是 PathTracer 類別中的 setFrameDim 函式。它用於設置渲染幀的尺寸，並檢查確保尺寸在支持的範圍內。

- 具體來說：

    1. 函式接受一個 uint2 型別的參數 frameDim，表示渲染幀的寬度和高度。
    2. 保存先前的渲染幀尺寸和屏幕磚數量，以便在新尺寸與先前尺寸不同時標記變化。
    3. 將 frameDim 賦值給 mParams.frameDim，即保存渲染幀尺寸的成員變數。
    4. 檢查新設置的渲染幀尺寸是否超出了支持的最大尺寸（kMaxFrameDimension）。如果超出，則拋出運行時錯誤。
    5. 確保磚的尺寸是2的冪次方，並且將磚的尺寸計算為 kScreenTileDim。磚的尺寸是渲染幀尺寸除以磚的數量。
    6. 檢查如果渲染幀尺寸或屏幕磚數量發生變化，則將 mVarsChanged 標記為 true。

- 總的來說，這個函式確保了渲染幀的尺寸在支持的範圍內，並計算了相應的屏幕磚數量。

```
void PathTracer::setScene(RenderContext* pRenderContext, const Scene::SharedPtr& pScene)
{
    mpScene = pScene;

    mParams.frameCount = 0;
    mParams.frameDim = {};
    mParams.screenTiles = {};

    // Need to recreate the RTXDI module when the scene changes.
    mpRTXDI = nullptr;
    mpConditionalReSTIRPass = nullptr;

    // Need to recreate the trace passes because the shader binding table changes.
    mpTracePass = nullptr;
    mpTraceDeltaReflectionPass = nullptr;
    mpTraceDeltaTransmissionPass = nullptr;
    mpGeneratePaths = nullptr;
    mpReflectTypes = nullptr;

    resetLighting();

    if (mpScene)
    {
        if (pScene->hasGeometryType(Scene::GeometryType::Custom))
        {
            logWarning("PathTracer: This render pass does not support custom primitives.");
        }

        validateOptions();

        mRecompile = true;
    }
}
```
- 這段程式碼是 PathTracer 類別中的 setScene 函式。這個函式用於設置要渲染的場景。

- 具體來說：

    - 將指向新場景的指標 pScene 賦值給成員變數 mpScene，從而更新要渲染的場景。
    - 將幀計數、渲染幀尺寸和屏幕磚數量重置為預設值。
    - 當場景改變時，需要重新創建 RTXDI 模組和 Conditional ReSTIR Pass。
    - 需要重新創建追蹤 Pass，因為着色器綁定表發生了變化。
    - 重置照明設置，這可能包括清除之前緩存的照明數據等操作。

- 如果新場景存在，則執行額外的操作：

    - 檢查場景是否包含自定義幾何類型，如果是則記錄警告，因為這個渲染 Pass 不支持自定義的幾何類型。
    - 驗證渲染選項，確保它們的合法性。
    - 將 mRecompile 標記為 true，表示需要重新編譯渲染程序。

- 總的來說，這個函式用於設置要渲染的場景，並相應地重新初始化和更新相關的內容，以確保渲染的正確性和一致性。

```cpp
void PathTracer::execute(RenderContext* pRenderContext, const RenderData& renderData)
{
    if (autoCompileMethods && !autoCompileFinished && mpConditionalReSTIRPass)
    {
        if (mRenderModePresetIdPrev != mRenderModePresetId)
            setPresetForMethod(mRenderModePresetId);

        mRenderModePresetIdPrev = mRenderModePresetId;

        if (mWarmupFramesSofar++ > 10) // the second frame
        {
            mWarmupFramesSofar = 0;
            if (mRenderModePresetId == 4)
            {
                autoCompileFinished = true;
                mRenderModePresetId = 0;
                setPresetForMethod(0);
            }
            else
            {
                mRenderModePresetId++;
            }
        }
    }

    if (!beginFrame(pRenderContext, renderData)) return;

    renderData.getDictionary()["freeze"] = mpScene->freeze;

    mUserInteractionRecorder.recordStep(mpScene);

    // Update shader program specialization.
    updatePrograms();

    // Prepare resources.
    prepareResources(pRenderContext, renderData);

    // Prepare the path tracer parameter block.
    // This should be called after all resources have been created.
    preparePathTracer(renderData);

    // Generate paths at primary hits.
    generatePaths(pRenderContext, renderData);

    // Update RTXDI.
    if (mpRTXDI && !mStaticParams.disableDirectIllumination && !mStaticParams.disableGeneralizedDirectIllumination)
    {
        const auto& pMotionVectors = renderData.getTexture(kInputMotionVectors);
        mpRTXDI->update(pRenderContext, pMotionVectors);
    }

    // loop spp times if ReSTIR is enabled

    // Launch separate passes to trace delta reflection and transmission paths to generate respective guide buffers.
    if (mOutputNRDAdditionalData)
    {
        FALCOR_ASSERT(mpTraceDeltaReflectionPass && mpTraceDeltaTransmissionPass);
        tracePass(pRenderContext, renderData, mpTraceDeltaReflectionPass);
        tracePass(pRenderContext, renderData, mpTraceDeltaTransmissionPass);
    }

    uint32_t iters = mParams.useConditionalReSTIR
                         ?  mParams.samplesPerPixel
                         : 1;

    for (uint32_t iter = 0; iter < iters; iter++)
    {
        // Trace pass.
        FALCOR_ASSERT(mpTracePass);
        tracePass(pRenderContext, renderData, mpTracePass, iter);
    }

    if (mpConditionalReSTIRPass && mParams.useConditionalReSTIR)
    {
        mpConditionalReSTIRPass->suffixResamplingPass(
            pRenderContext, renderData.getTexture(kInputVBuffer),
            renderData.getTexture(kInputMotionVectors),
            renderData.getTexture(kOutputColor));
    }

    // Resolve pass.
    resolvePass(pRenderContext, renderData);

    endFrame(pRenderContext, renderData);

    mIsFrozen = mpScene->freeze;

    if (!mpScene->freeze)
    {
        bool shouldFreeze = mUserInteractionRecorder.replayStep(mpScene);
        if (shouldFreeze)
        {
            mpScene->freeze = true;
        }
    }
}
```
- 這段程式碼是 PathTracer 類別中的 execute 函式。這個函式用於執行渲染操作。具體來說：

    - 如果 autoCompileMethods 為真且 autoCompileFinished 為假且 mpConditionalReSTIRPass 不為空，則執行自動編譯過程。
    - 如果不是新的一幀，則更新渲染模式並準備下一個自動編譯過程。
    - 如果開始渲染一幀不成功，則退出函式。
    - 記錄場景是否被凍結。
    - 記錄用戶交互記錄。
    - 更新着色器程序的特化。
    - 準備資源。
    - 準備 PathTracer 參數塊。
    - 在主要命中點生成路徑。
    - 更新 RTXDI。
    - 迴圈執行 samplesPerPixel 次數，如果使用 Conditional ReSTIR 則迴圈執行一次。
    - 分別啟動追蹤通過來追蹤增量反射和傳輸路徑以生成相應的引導緩衝區。
    - 如果啟用 Conditional ReSTIR，則執行條件 ReSTIR Pass 的後綴重樣本過程。
    - 解析 Pass。
    - 結束渲染一幀。
    - 檢查場景是否被凍結，並根據用戶交互記錄來決定是否應該凍結場景。

- 總的來說，這個函式用於執行一幀的渲染過程，包括準備資源、生成路徑、追蹤路徑、解析結果等操作，同時也處理了自動編譯和用戶交互的相關邏輯。

```cpp
void PathTracer::renderUI(Gui::Widgets& widget)
{
    bool dirty = false;

    auto group = widget.group("User Interaction Recording", false);

    dirty |= mUserInteractionRecorder.renderUI(group);

    if (auto group = widget.group("Rendering Presets", true))
    {
        // method selection
        bool isRenderModeChanged = widget.dropdown("Render Mode Preset", kRenderModePreset, mRenderModePresetId);
        if (isRenderModeChanged && mpConditionalReSTIRPass)
        {
            dirty = true;

            setPresetForMethod(mRenderModePresetId);
        }

        if (mpConditionalReSTIRPass && mParams.useConditionalReSTIR)
        {
            bool changed = widget.var("Num Integration Prefixes", mpConditionalReSTIRPass->getOptions().subpathSetting.numIntegrationPrefixes, 1, 128);
            bool needReallocate = widget.var("Final Gather Suffixes", mpConditionalReSTIRPass->getOptions().subpathSetting.finalGatherSuffixCount, 1, 8);

            if (changed || needReallocate)
            {
                dirty = true;
                mpConditionalReSTIRPass->mResetTemporalReservoirs = true;
                if (needReallocate)
                {
                    mpConditionalReSTIRPass->mReallocate = true;
                    mpConditionalReSTIRPass->mRecompile = true;
                }
            }
        }
    }

    // Rendering options.
    dirty |= renderRenderingUI(widget);

    // Stats and debug options.
    renderStatsUI(widget);
    dirty |= renderDebugUI(widget);

    if (dirty)
    {
        validateOptions();
        mOptionsChanged = true;
    }
}
```
- 這段程式碼是 PathTracer 類別中的 renderUI 函式。這個函式用於渲染用戶界面（UI），並允許用戶進行相關設置。具體來說：

    - 創建一個布爾變數 dirty 來跟蹤界面是否需要重新渲染。
    - 創建一個包含用戶交互記錄的組件組 group，並呼叫該組件的 renderUI 函式以渲染用戶交互記錄的相關內容，如果內容有變動，則將 dirty 設置為 true。
    - 如果存在名為 "Rendering Presets" 的組件組，則：
        - 渲染渲染模式預設下拉菜單，允許用戶選擇渲染模式預設，如果選擇發生變化且 mpConditionalReSTIRPass 存在，則設置 dirty 為 true，並根據選擇設置渲染模式。
        - 如果 mpConditionalReSTIRPass 存在且使用 Conditional ReSTIR，則渲染用於設置集成前綴數量和最終收集後綴數量的組件，如果設置發生變化，則設置 dirty 為 true，並標記需要重新分配資源和重新編譯。
    - 渲染渲染選項。
    - 渲染統計信息和調試選項的界面。
    - 如果 dirty 為 true，則驗證渲染選項並將 mOptionsChanged 設置為 true，表示選項已更改。

- 總的來說，這個函式負責渲染 PathTracer 的用戶界面，並處理用戶的相關操作，例如選擇渲染模式預設、設置集成前綴和最終收集後綴數量，以及渲染其他渲染和調試選項。

```cpp
bool PathTracer::renderRenderingUI(Gui::Widgets& widget)
{
    bool dirty = false;
    bool runtimeDirty = false;

    if (auto group = widget.group("Path Tracer Options", false))
    {
        if (mFixedSampleCount)
        {
            dirty |= widget.var("Samples/pixel", mParams.samplesPerPixel, 1, (int)kMaxSamplesPerPixel);
        }
        else widget.text("Samples/pixel: Variable");
        widget.tooltip("Number of samples per pixel. One path is traced for each sample.\n\n"
            "When the '" + kInputSampleCount + "' input is connected, the number of samples per pixel is loaded from the texture.");

        if (widget.var("Max surface bounces", mStaticParams.maxSurfaceBounces, 0u, kMaxBounces))
        {
            // Allow users to change the max surface bounce parameter in the UI to clamp all other surface bounce parameters.
            mStaticParams.maxDiffuseBounces = std::min(mStaticParams.maxDiffuseBounces, mStaticParams.maxSurfaceBounces);
            mStaticParams.maxSpecularBounces = std::min(mStaticParams.maxSpecularBounces, mStaticParams.maxSurfaceBounces);
            mStaticParams.maxTransmissionBounces = std::min(mStaticParams.maxTransmissionBounces, mStaticParams.maxSurfaceBounces);
            dirty = true;
        }
        widget.tooltip("Maximum number of surface bounces (diffuse + specular + transmission).\n"
            "Note that specular reflection events from a material with a roughness greater than specularRoughnessThreshold are also classified as diffuse events.");

        dirty |= widget.var("Max diffuse bounces", mStaticParams.maxDiffuseBounces, 0u, kMaxBounces);
        widget.tooltip("Maximum number of diffuse bounces.\n0 = direct only\n1 = one indirect bounce etc.");

        dirty |= widget.var("Max specular bounces", mStaticParams.maxSpecularBounces, 0u, kMaxBounces);
        widget.tooltip("Maximum number of specular bounces.\n0 = direct only\n1 = one indirect bounce etc.");

        dirty |= widget.var("Max transmission bounces", mStaticParams.maxTransmissionBounces, 0u, kMaxBounces);
        widget.tooltip("Maximum number of transmission bounces.\n0 = no transmission\n1 = one transmission bounce etc.");

        // Sampling options.

        if (widget.dropdown("Sample generator", SampleGenerator::getGuiDropdownList(), mStaticParams.sampleGenerator))
        {
            mpSampleGenerator = SampleGenerator::create(mStaticParams.sampleGenerator);
            dirty = true;
        }

        dirty |= widget.checkbox("BSDF importance sampling", mStaticParams.useBSDFSampling);
        widget.tooltip("BSDF importance sampling should normally be enabled.\n\n"
            "If disabled, cosine-weighted hemisphere sampling is used for debugging purposes");

        dirty |= widget.checkbox("Russian roulette", mStaticParams.useRussianRoulette);
        widget.tooltip("Use russian roulette to terminate low throughput paths.");

        dirty |= widget.checkbox("Next-event estimation (NEE)", mStaticParams.useNEE);
        widget.tooltip("Use next-event estimation.\nThis option enables direct illumination sampling at each path vertex.");

        if (mStaticParams.useNEE)
        {
            dirty |= widget.checkbox("Multiple importance sampling (MIS)", mStaticParams.useMIS);
            widget.tooltip("When enabled, BSDF sampling is combined with light sampling for the environment map and emissive lights.\n"
                "Note that MIS has currently no effect on analytic lights.");

            if (mStaticParams.useMIS)
            {
                dirty |= widget.dropdown("MIS heuristic", kMISHeuristicList, reinterpret_cast<uint32_t&>(mStaticParams.misHeuristic));

                if (mStaticParams.misHeuristic == MISHeuristic::PowerExp)
                {
                    dirty |= widget.var("MIS power exponent", mStaticParams.misPowerExponent, 0.01f, 10.f);
                }
            }

            if (mpScene && mpScene->useEmissiveLights())
            {
                if (auto group = widget.group("Emissive sampler"))
                {
                    if (widget.dropdown("Emissive sampler", kEmissiveSamplerList, (uint32_t&)mStaticParams.emissiveSampler))
                    {
                        resetLighting();
                        dirty = true;
                    }
                    widget.tooltip("Selects which light sampler to use for importance sampling of emissive geometry.", true);

                    if (mpEmissiveSampler)
                    {
                        if (mpEmissiveSampler->renderUI(group)) mOptionsChanged = true;
                    }
                }
            }
        }
    }

    if (auto group = widget.group("RTXDI"))
    {
        dirty |= widget.checkbox("Enabled", mStaticParams.useRTXDI);
        widget.tooltip("Use RTXDI for direct illumination.");
        if (mpRTXDI) dirty |= mpRTXDI->renderUI(group);
    }

    if (auto group = widget.group("Conditional ReSTIR"))
    {
        dirty |= widget.checkbox("Enabled", mParams.useConditionalReSTIR);
        widget.tooltip("Use Conditional ReSTIR (Final Gather version of ReSTIR PT) for indirect illumination.");
        if (mpConditionalReSTIRPass) dirty |= mpConditionalReSTIRPass->renderUI(group);
    }

    if (auto group = widget.group("Material controls"))
    {
        dirty |= widget.checkbox("Alpha test", mStaticParams.useAlphaTest);
        widget.tooltip("Use alpha testing on non-opaque triangles.");

        dirty |= widget.checkbox("Adjust shading normals on secondary hits", mStaticParams.adjustShadingNormals);
        widget.tooltip("Enables adjustment of the shading normals to reduce the risk of black pixels due to back-facing vectors.\nDoes not apply to primary hits which is configured in GBuffer.", true);

        dirty |= widget.var("Max nested materials", mStaticParams.maxNestedMaterials, 2u, 4u);
        widget.tooltip("Maximum supported number of nested materials.");

        dirty |= widget.checkbox("Use lights in dielectric volumes", mStaticParams.useLightsInDielectricVolumes);
        widget.tooltip("Use lights inside of volumes (transmissive materials). We typically don't want this because lights are occluded by the interface.");

        dirty |= widget.checkbox("Disable caustics", mStaticParams.disableCaustics);
        widget.tooltip("Disable sampling of caustic light paths (i.e. specular events after diffuse events).");

        runtimeDirty |= widget.var("Specular roughness threshold", mParams.specularRoughnessThreshold, 0.f, 1.f);
        widget.tooltip("Specular reflection events are only classified as specular if the material's roughness value is equal or smaller than this threshold. Otherwise they are classified diffuse.");

        dirty |= widget.dropdown("Primary LOD Mode", kLODModeList, reinterpret_cast<uint32_t&>(mStaticParams.primaryLodMode));
        widget.tooltip("Texture LOD mode at primary hit");

        runtimeDirty |= widget.var("TexLOD bias", mParams.lodBias, -16.f, 16.f, 0.01f);

        dirty |= widget.checkbox("Use Lambertian Diffuse", mStaticParams.useLambertianDiffuse);
        widget.tooltip("Use the simpler Lambertian model for diffuse reflection");

        dirty |= widget.dropdown("DI Mode", kDIMode, reinterpret_cast<uint32_t&>(mParams.DIMode));

        dirty |= widget.checkbox("Disable Diffuse", mStaticParams.disableDiffuse);

        dirty |= widget.checkbox("Disable Specular", mStaticParams.disableSpecular);

        dirty |= widget.checkbox("Disable Translucency", mStaticParams.disableTranslucency);
    }

    if (auto group = widget.group("Denoiser options"))
    {
        dirty |= widget.checkbox("Use NRD demodulation", mStaticParams.useNRDDemodulation);
        widget.tooltip("Global switch for NRD demodulation");
    }

    if (auto group = widget.group("Output options"))
    {
        // Switch to enable/disable path tracer output.
        dirty |= widget.checkbox("Enable output", mEnabled);

        // Controls for output size.
        // When output size requirements change, we'll trigger a graph recompile to update the render pass I/O sizes.
        if (widget.dropdown("Output size", RenderPassHelpers::kIOSizeList, (uint32_t&)mOutputSizeSelection)) requestRecompile();
        if (mOutputSizeSelection == RenderPassHelpers::IOSize::Fixed)
        {
            if (widget.var("Size in pixels", mFixedOutputSize, 32u, 16384u)) requestRecompile();
        }

        dirty |= widget.dropdown("Color format", kColorFormatList, (uint32_t&)mStaticParams.colorFormat);
        widget.tooltip("Selects the color format used for internal per-sample color and denoiser buffers");
    }

    if (dirty) mRecompile = true;
    return dirty || runtimeDirty;
}
```
- 這段程式碼是 PathTracer 類別中的 renderRenderingUI 函式。這個函式負責渲染與渲染相關的用戶界面（UI），並處理用戶的相關操作。具體來說：

- 創建兩個布爾變數 dirty 和 runtimeDirty 來跟蹤界面是否需要重新渲染。
- 如果存在名為 "Path Tracer Options" 的組件組，則渲染以下選項：
    - "Samples/pixel"：渲染每個像素的樣本數選項。
    - "Max surface bounces"、"Max diffuse bounces"、"Max specular bounces" 和 "Max transmission bounces"：最大表面反射、漫反射、鏡面反射和透射反射次數選項。
    - "Sample generator"：樣本生成器選擇。
    - "BSDF importance sampling"、"Russian roulette"、"Next-event estimation (NEE)" 和 "Multiple importance sampling (MIS)"：BSDF 重要性取樣、俄羅斯輪盤、下一事件估計和多重重要性取樣選項。
    - "Emissive sampler"：發射光取樣器選擇。
- 如果存在名為 "RTXDI" 的組件組，則渲染用於直接光照的 RTXDI 選項。
- 如果存在名為 "Conditional ReSTIR" 的組件組，則渲染用於間接光照的 Conditional ReSTIR 選項。
- 如果存在名為 "Material controls" 的組件組，則渲染與材質相關的控制選項。
- 如果存在名為 "Denoiser options" 的組件組，則渲染去噪器相關的選項。
- 如果存在名為 "Output options" 的組件組，則渲染與輸出相關的選項，包括輸出啟用開關、輸出尺寸選項和顏色格式選項。
- 如果任何選項被更改，則將 dirty 或 runtimeDirty 設置為 true，表示需要重新編譯。

- 總的來說，這個函式負責渲染 PathTracer 的渲染選項和控制界面，並處理用戶對這些選項的操作。

```cpp
bool PathTracer::renderDebugUI(Gui::Widgets& widget)
{
    bool dirty = false;

    if (auto group = widget.group("Debugging"))
    {
        dirty |= group.checkbox("Use fixed seed", mParams.useFixedSeed);
        group.tooltip("Forces a fixed random seed for each frame.\n\n"
            "This should produce exactly the same image each frame, which can be useful for debugging.");
        if (mParams.useFixedSeed)
        {
            dirty |= group.var("Seed", mParams.fixedSeed);
        }

        mpPixelDebug->renderUI(group);
    }

    return dirty;
}
```
- 這段程式碼是 PathTracer 類別中的 renderDebugUI 函式。這個函式用於渲染調試相關的用戶界面，並處理與調試相關的選項。具體來說：

- 如果存在名為 "Debugging" 的組件組，則渲染以下選項：
    - "Use fixed seed" 复选框：用於控制是否使用固定的隨機種子。
    - 如果 "Use fixed seed" 被選中，則顯示 "Seed" 變數，允許用戶指定固定的種子值。
    - 調用 mpPixelDebug 對象的 renderUI 函式，渲染與像素調試相關的選項。
- 如果任何選項被更改，則將 dirty 設置為 true，表示需要更新界面。

- 總的來說，這個函式用於渲染 PathTracer 的調試選項和控制界面，並處理用戶對這些選項的操作。

```cpp
void PathTracer::renderStatsUI(Gui::Widgets& widget)
{
    if (auto g = widget.group("Statistics"))
    {
        // Show ray stats
        mpPixelStats->renderUI(g);
    }
}
```
- 這段程式碼是 PathTracer 類別中的 renderStatsUI 函式。這個函式用於渲染統計信息的用戶界面，主要顯示與渲染過程中生成的統計數據相關的信息。具體來說：

- 如果存在名為 "Statistics" 的組件組，則渲染以下統計信息：
    - 調用 mpPixelStats 對象的 renderUI 函式，用於渲染與像素統計相關的統計信息。

- 總的來說，這個函式用於渲染 PathTracer 的統計信息的用戶界面，並通過調用對應的統計對象的 renderUI 函式來實現。

```cpp
bool PathTracer::onMouseEvent(const MouseEvent& mouseEvent)
{
    bool dirty = mpPixelDebug->onMouseEvent(mouseEvent);
    if (mpConditionalReSTIRPass) dirty |= mpConditionalReSTIRPass->getPixelDebug()->onMouseEvent(mouseEvent);
    return dirty;
}
```
- 這段程式碼是 PathTracer 類別中的 onMouseEvent 函式。該函式用於處理滑鼠事件，並返回一個布林值，表示是否有狀態變化需要更新。

- 具體來說：

    - 呼叫 mpPixelDebug 物件的 onMouseEvent 函式來處理滑鼠事件，並將返回的結果賦值給變數 dirty。
    - 如果存在 mpConditionalReSTIRPass 物件，則呼叫其 getPixelDebug 函式獲取像素調試物件，並呼叫其 onMouseEvent 函式處理滑鼠事件，然後將返回的結果與之前的 dirty 變數進行邏輯或操作。
    - 最終返回 dirty 變數，表示是否有狀態變化需要更新。

```cpp
void PathTracer::setModeId(int modeId)
{
    mRenderModePresetId = modeId;
    setPresetForMethod(mRenderModePresetId, false);
}
```
- 這段程式碼是 PathTracer 類別中的 setModeId 函式。該函式用於設置渲染模式的 ID。

- 具體來說：

    - 將傳入的渲染模式 ID（modeId）賦值給成員變數 mRenderModePresetId。
    - 調用 setPresetForMethod 函式，將 mRenderModePresetId 作為參數傳遞給它，以設置相應的渲染模式。

```cpp
void PathTracer::updateDict(const Dictionary& dict)
{
    parseDictionary(dict);
    if (mpConditionalReSTIRPass)
    {
        mpConditionalReSTIRPass->setOptions(mConditionalReSTIROptions);
        mpConditionalReSTIRPass->mReallocate = true;
        mpConditionalReSTIRPass->mRecompile = true;
        mpConditionalReSTIRPass->mResetTemporalReservoirs = true;
    }
    mRecompile = true;
    mOptionsChanged = true;
}
```
- 這段程式碼是 PathTracer 類別中的 updateDict 函式。該函式用於更新字典中的參數。

- 具體來說：

    - 解析傳入的字典（dict）。
    - 如果存在 mpConditionalReSTIRPass，則將 mConditionalReSTIROptions 設置為其選項。同時，設置 mpConditionalReSTIRPass 的重新分配標誌、重新編譯標誌和重置時間蓄水池的標誌為 true。
    - 將 mRecompile 和 mOptionsChanged 設置為 true，表示需要重新編譯渲染程序和選項已更改。

```cpp
void PathTracer::updatePrograms()
{
    FALCOR_ASSERT(mpScene);

    if (mRecompile == false) return;

    auto defines = mStaticParams.getDefines(*this);
    auto globalTypeConformances = mpScene->getMaterialSystem()->getTypeConformances();

    // Create compute passes.
    Program::Desc baseDesc;
    baseDesc.addShaderModules(mpScene->getShaderModules());
    baseDesc.addTypeConformances(globalTypeConformances);
    baseDesc.setShaderModel(kShaderModel);

    if (!mpTracePass)
    {
        Program::Desc desc = baseDesc;
        desc.addShaderLibrary(kTracePassFilename).csEntry("main");
        mpTracePass = ComputePass::create(desc, defines, false);
    }

    if (mOutputNRDAdditionalData && (!mpTraceDeltaReflectionPass || !mpTraceDeltaTransmissionPass))
    {
        Program::Desc desc = baseDesc;
        desc.addShaderLibrary(kTracePassFilename).csEntry("main");

        Program::DefineList deltaReflectionTraceDefine = defines;
        deltaReflectionTraceDefine.add("DELTA_REFLECTION_PASS");
        mpTraceDeltaReflectionPass = ComputePass::create(desc, deltaReflectionTraceDefine, false);

        Program::DefineList deltaTransmissionTraceDefine = defines;
        deltaTransmissionTraceDefine.add("DELTA_TRANSMISSION_PASS");
        mpTraceDeltaTransmissionPass = ComputePass::create(desc, deltaTransmissionTraceDefine, false);
    }

    if (!mpGeneratePaths)
    {
        Program::Desc desc = baseDesc;
        desc.addShaderLibrary(kGeneratePathsFilename).csEntry("main");
        mpGeneratePaths = ComputePass::create(desc, defines, false);
    }
    if (!mpReflectTypes)
    {
        Program::Desc desc = baseDesc;
        desc.addShaderLibrary(kReflectTypesFile).csEntry("main");
        mpReflectTypes = ComputePass::create(desc, defines, false);
    }

    // Perform program specialization.
    // Note that we must use set instead of add functions to replace any stale state.
    auto prepareProgram = [&](Program::SharedPtr program)
    {
        program->setDefines(defines);
    };
    prepareProgram(mpTracePass->getProgram());
    prepareProgram(mpGeneratePaths->getProgram());
    prepareProgram(mpResolvePass->getProgram());
    prepareProgram(mpReflectTypes->getProgram());

    // Create program vars for the specialized programs.
    mpTracePass->setVars(nullptr);
    if (mpTraceDeltaReflectionPass && mpTraceDeltaTransmissionPass)
    {
        mpTraceDeltaReflectionPass->setVars(nullptr);
        mpTraceDeltaTransmissionPass->setVars(nullptr);
    }
    mpGeneratePaths->setVars(nullptr);
    mpResolvePass->setVars(nullptr);
    mpReflectTypes->setVars(nullptr);

    mVarsChanged = true;
    mRecompile = false;

    // since ReSTIR shares some macro definition with the host program, we need to update as well
    if (mpConditionalReSTIRPass) mpConditionalReSTIRPass->updatePrograms();
}
```
- 這段程式碼是 PathTracer 類別中的 updatePrograms 函式。該函式用於更新計算通過的程序。

- 具體來說：

    - 首先，檢查是否需要重新編譯。如果不需要，則直接返回，不進行任何操作。
    - 根據場景和靜態參數獲取定義。
    - 獲取全局類型兼容性。
    - 創建計算通過。這包括追蹤通過（mpTracePass）、增量反射通過（mpTraceDeltaReflectionPass）、增量透射通過（mpTraceDeltaTransmissionPass）、生成路徑通過（mpGeneratePaths）和反射類型通過（mpReflectTypes）。
    - 執行程序特化。通過設置相應的定義，將通過特化為具體的需求。
    - 更新計算通過的程序變量（ProgramVars）。
    - 將 mVarsChanged 和 mRecompile 設置為 true，表示程序變量已更改並且需要重新編譯。
    - 如果存在 mpConditionalReSTIRPass，則更新其程序。

- 總的來說，這個函式用於更新計算通過的程序，確保它們與當前的場景和參數設置相匹配。

```cpp
void PathTracer::prepareResources(RenderContext* pRenderContext, const RenderData& renderData)
{
    // Compute allocation requirements for paths and output samples.
    // Note that the sample buffers are padded to whole tiles, while the max path count depends on actual frame dimension.
    // If we don't have a fixed sample count, assume the worst case.

    if (mOutputGuideData || mOutputNRDData) mParams.samplesPerPixel = std::min(mParams.samplesPerPixel, 16); // avoid creating large buffer
    uint32_t spp = mFixedSampleCount ? mParams.samplesPerPixel : kMaxSamplesPerPixel;
    uint32_t tileCount = mParams.screenTiles.x * mParams.screenTiles.y;
    const uint32_t sampleCount = tileCount * kScreenTileDim.x * kScreenTileDim.y * spp;
    const uint32_t screenPixelCount = mParams.frameDim.x * mParams.frameDim.y;
    const uint32_t pathCount = screenPixelCount * spp;

    // Allocate output sample offset buffer if needed.
    // This buffer stores the output offset to where the samples for each pixel are stored consecutively.
    // The offsets are local to the current tile, so 16-bit format is sufficient and reduces bandwidth usage.
    if (!mFixedSampleCount)
    {
        if (!mpSampleOffset || mpSampleOffset->getWidth() != mParams.frameDim.x || mpSampleOffset->getHeight() != mParams.frameDim.y)
        {
            FALCOR_ASSERT(kScreenTileDim.x * kScreenTileDim.y * kMaxSamplesPerPixel <= (1u << 16));
            mpSampleOffset = Texture::create2D(mParams.frameDim.x, mParams.frameDim.y, ResourceFormat::R16Uint, 1, 1, nullptr, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess);
            mVarsChanged = true;
        }
    }

    auto var = mpReflectTypes->getRootVar();

    if (mOutputGuideData && (!mpSampleGuideData || mpSampleGuideData->getElementCount() < sampleCount || mVarsChanged))
    {
        mpSampleGuideData = Buffer::createStructured(var["sampleGuideData"], sampleCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        mVarsChanged = true;
    }

    if (mOutputNRDData && (!mpSampleNRDRadiance || mpSampleNRDRadiance->getElementCount() < sampleCount || mVarsChanged))
    {
        mpSampleNRDRadiance = Buffer::createStructured(var["sampleNRDRadiance"], sampleCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        mpSampleNRDHitDist = Buffer::createStructured(var["sampleNRDHitDist"], sampleCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        mpSampleNRDPrimaryHitNeeOnDelta = Buffer::createStructured(var["sampleNRDPrimaryHitNeeOnDelta"], sampleCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        mpSampleNRDEmission = Buffer::createStructured(var["sampleNRDEmission"], sampleCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        mpSampleNRDPrimaryHitEmission = Buffer::createStructured(var["sampleNRDEmission"], sampleCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        mpSampleNRDReflectance = Buffer::createStructured(var["sampleNRDReflectance"], sampleCount, Resource::BindFlags::ShaderResource | Resource::BindFlags::UnorderedAccess, Buffer::CpuAccess::None, nullptr, false);
        mVarsChanged = true;
    }
}
```
- 這段程式碼是 PathTracer 類別中的 prepareResources 函式。該函式用於準備計算所需的資源，包括輸出樣本和路徑的分配需求。

- 具體來說：

- 首先，根據輸出指南資料和非透射數據的需要，調整每像素的樣本數。如果輸出指南資料或非透射數據的需求存在，且每像素樣本數大於16，則將每像素樣本數調整為16，以避免創建過大的緩衝區。
- 然後計算需要的緩衝區大小。這包括每個像素的樣本數（sampleCount）和路徑數（pathCount）。
- 接著根據需要分配緩衝區。具體來說：
    - 如果樣本數不固定，則分配輸出樣本偏移緩衝區（mpSampleOffset），其存儲了每個像素樣本的偏移量。
    - 如果需要輸出指南資料，則分配相應的緩衝區（mpSampleGuideData）。
    - 如果需要非透射數據，則分配相應的緩衝區，包括散射的輻射（mpSampleNRDRadiance）、散射的距離（mpSampleNRDHitDist）、非透射主要命中 NEE On Delta（mpSampleNRDPrimaryHitNeeOnDelta）、散射的發射（mpSampleNRDEmission）、主要命中發射（mpSampleNRDPrimaryHitEmission）和散射的反射率（mpSampleNRDReflectance）。

- 總的來說，這個函式用於準備計算所需的資源，以確保計算能夠正常運行並生成所需的輸出資料。

```cpp
void PathTracer::preparePathTracer(const RenderData& renderData)
{
    // Create path tracer parameter block if needed.
    if (!mpPathTracerBlock || mVarsChanged)
    {
        auto reflector = mpReflectTypes->getProgram()->getReflector()->getParameterBlock("pathTracer");
        mpPathTracerBlock = ParameterBlock::create(reflector);
        FALCOR_ASSERT(mpPathTracerBlock);
        mVarsChanged = true;
    }

    // Bind resources.
    auto var = mpPathTracerBlock->getRootVar();
    setShaderData(var, renderData);

    // set path tracer shader data for ReSTIR
    if (mParams.useConditionalReSTIR && mpConditionalReSTIRPass)
    {
        if (mVarsChanged || !mpConditionalReSTIRPass->getPathTracerBlock())
            mpConditionalReSTIRPass->createPathTracerBlock();

        auto var = mpConditionalReSTIRPass->getPathTracerBlock()->getRootVar();
        setPathTracerDataForConditionalReSTIR(var, renderData);
    }
}
```
- 這段程式碼是 PathTracer 類別中的 preparePathTracer 函式。該函式用於準備路徑追踪器所需的參數塊以及綁定資源。

- 具體來說：

    - 首先，檢查是否需要創建路徑追踪器的參數塊（mpPathTracerBlock）或者參數塊是否已經被修改（mVarsChanged）。如果需要創建或者參數塊已被修改，則根據反射型別（reflector）創建參數塊。
    - 接著，綁定資源到參數塊中。根據傳入的渲染資料（renderData），設置Shader Data到參數塊的根變數中。
    - 如果使用了條件式 ReSTIR 且存在 mpConditionalReSTIRPass，則設置路徑追踪器的 Shader Data。在此情況下，同樣檢查是否需要創建路徑追踪器的參數塊，並根據需要創建或者已修改的情況進行處理。

- 總的來說，這個函式用於準備路徑追踪器所需的參數塊，並綁定相應的資源，以確保路徑追踪計算能夠正確執行並獲取所需的資料。

```cpp
void PathTracer::resetLighting()
{
    // Retain the options for the emissive sampler.
    if (auto lightBVHSampler = std::dynamic_pointer_cast<LightBVHSampler>(mpEmissiveSampler))
    {
        mLightBVHOptions = lightBVHSampler->getOptions();
    }

    mpEmissiveSampler = nullptr;
    mpEnvMapSampler = nullptr;
    mRecompile = true;
}
```
- 這段程式碼是 PathTracer 類別中的 resetLighting 函式。該函式用於重置光照相關的設置。

- 具體來說：

    - 首先，保存發光物體取樣器（mpEmissiveSampler）的選項，以便稍後重置時可以保留這些選項。如果目前的發光物體取樣器是 LightBVHSampler 的話，則保存其選項（mLightBVHOptions）。
    - 然後，將發光物體取樣器和環境貼圖取樣器（mpEnvMapSampler）設置為空指針，以清除這些光照相關的設置。
    - 最後，將重新編譯標誌（mRecompile）設置為 true，以表示需要重新編譯相關的程序。

- 總的來說，這個函式用於重置光照相關的設置，並準備進行相應的重建工作，以確保後續的光照計算能夠正確執行。

```cpp
void PathTracer::prepareMaterials(RenderContext* pRenderContext)
{
    // This functions checks for material changes and performs any necessary update.
    // For now all we need to do is to trigger a recompile so that the right defines get set.
    // In the future, we might want to do additional material-specific setup here.

    if (is_set(mpScene->getUpdates(), Scene::UpdateFlags::MaterialsChanged))
    {
        mRecompile = true;
    }
}
```
- 這段程式碼是 PathTracer 類別中的 prepareMaterials 函式。該函式用於準備材質相關的設置。

- 具體來說：

    - 此函式檢查材質是否發生了變化（通過檢查場景的更新標誌）。如果材質發生了變化，則設置重新編譯標誌（mRecompile）為 true。
    - 目前，這個函式僅僅觸發重新編譯，以確保正確的宏定義被設置。在將來，可能會在這裡進行額外的材質特定的設置。

- 總的來說，這個函式用於檢查並處理材質相關的變化，以確保後續的渲染工作能夠正確進行。

```cpp
bool PathTracer::prepareLighting(RenderContext* pRenderContext)
{
    bool lightingChanged = false;

    if (is_set(mpScene->getUpdates(), Scene::UpdateFlags::RenderSettingsChanged))
    {
        lightingChanged = true;
        mRecompile = true;
    }

    if (is_set(mpScene->getUpdates(), Scene::UpdateFlags::SDFGridConfigChanged))
    {
        mRecompile = true;
    }

    if (is_set(mpScene->getUpdates(), Scene::UpdateFlags::EnvMapChanged))
    {
        mpEnvMapSampler = nullptr;
        lightingChanged = true;
        mRecompile = true;
    }

    if (mpScene->useEnvLight())
    {
        if (!mpEnvMapSampler)
        {
            mpEnvMapSampler = EnvMapSampler::create(pRenderContext, mpScene->getEnvMap());
            lightingChanged = true;
            mRecompile = true;
        }
    }
    else
    {
        if (mpEnvMapSampler)
        {
            mpEnvMapSampler = nullptr;
            lightingChanged = true;
            mRecompile = true;
        }
    }

    // Request the light collection if emissive lights are enabled.
    if (mpScene->getRenderSettings().useEmissiveLights)
    {
        mpScene->getLightCollection(pRenderContext);
    }

    if (mpScene->useEmissiveLights())
    {
        if (!mpEmissiveSampler)
        {
            const auto& pLights = mpScene->getLightCollection(pRenderContext);
            FALCOR_ASSERT(pLights && pLights->getActiveLightCount() > 0);
            FALCOR_ASSERT(!mpEmissiveSampler);

            switch (mStaticParams.emissiveSampler)
            {
            case EmissiveLightSamplerType::Uniform:
                mpEmissiveSampler = EmissiveUniformSampler::create(pRenderContext, mpScene);
                break;
            case EmissiveLightSamplerType::LightBVH:
                mpEmissiveSampler = LightBVHSampler::create(pRenderContext, mpScene, mLightBVHOptions);
                break;
            case EmissiveLightSamplerType::Power:
                mpEmissiveSampler = EmissivePowerSampler::create(pRenderContext, mpScene);
                break;
            default:
                throw RuntimeError("Unknown emissive light sampler type");
            }
            lightingChanged = true;
            mRecompile = true;
        }
    }
    else
    {
        if (mpEmissiveSampler)
        {
            // Retain the options for the emissive sampler.
            if (auto lightBVHSampler = std::dynamic_pointer_cast<LightBVHSampler>(mpEmissiveSampler))
            {
                mLightBVHOptions = lightBVHSampler->getOptions();
            }

            mpEmissiveSampler = nullptr;
            lightingChanged = true;
            mRecompile = true;
        }
    }

    if (mpEmissiveSampler)
    {
        lightingChanged |= mpEmissiveSampler->update(pRenderContext);
        auto defines = mpEmissiveSampler->getDefines();
        if (mpTracePass && mpTracePass->getProgram()->addDefines(defines)) mRecompile = true;
    }

    return lightingChanged;
}
```
- 這段程式碼是 PathTracer 類別中的 prepareLighting 函式。該函式用於準備照明相關的設置。

- 具體來說：

    - 函式首先檢查場景是否有更新，包括渲染設置、SDF 網格配置和環境貼圖。如果有任何變化，則設置重新編譯標誌（mRecompile）為 true。
    - 如果環境貼圖發生變化，則將環境貼圖取樣器（mpEnvMapSampler）設置為 nullptr，並設置重新編譯標誌為 true。
    - 如果場景使用環境光，且環境貼圖可用，則創建並設置環境貼圖取樣器。
    - 如果場景中使用了發光物體，則根據所選的發光物體取樣器類型創建並設置對應的發光物體取樣器（mpEmissiveSampler）。
    - 最後，根據是否有發光物體取樣器，更新取樣器並根據取樣器定義更新程式。

- 總的來說，這個函式用於準備照明相關的設置，包括環境貼圖和發光物體的處理。

```cpp
void PathTracer::prepareRTXDI(RenderContext* pRenderContext)
{
    if (mStaticParams.useRTXDI)
    {
        if (!mpRTXDI) mpRTXDI = RTXDI::create(mpScene, mRTXDIOptions);
    }
    else
    {
        mpRTXDI = nullptr;
    }
}
```
- 這段程式碼是 PathTracer 類別中的一部分，負責準備實時光線追蹤直接光照 (RTXDI) 所需的資源和設定。

1. 首先檢查靜態參數 useRTXDI 是否為真，表示是否要使用 RTXDI。如果是，則執行以下操作：
    - 檢查 mpRTXDI 是否為空指針，即是否已經創建了 RTXDI 實例。
    - 如果 mpRTXDI 是空指針，則使用 RTXDI::create 函數創建一個新的 RTXDI 實例，該函數需要傳入場景 mpScene 和 RTXDI 選項 mRTXDIOptions。

2. 如果 useRTXDI 為假，即不使用 RTXDI，則將 mpRTXDI 設置為空指針。

- 簡而言之，這段程式碼的作用是根據靜態參數來決定是否使用 RTXDI，並根據需要創建或銷毀 RTXDI 實例。

```cpp
void PathTracer::setNRDData(const ShaderVar& var, const RenderData& renderData) const
{
    var["sampleRadiance"] = mpSampleNRDRadiance;
    var["sampleHitDist"] = mpSampleNRDHitDist;
    var["samplePrimaryHitNEEOnDelta"] = mpSampleNRDPrimaryHitNeeOnDelta;
    var["sampleEmission"] = mpSampleNRDEmission;
    var["samplePrimaryHitEmission"] = mpSampleNRDPrimaryHitEmission;
    var["sampleReflectance"] = mpSampleNRDReflectance;
    var["primaryHitEmission"] = renderData.getTexture(kOutputNRDEmission);
    var["primaryHitDiffuseReflectance"] = renderData.getTexture(kOutputNRDDiffuseReflectance);
    var["primaryHitSpecularReflectance"] = renderData.getTexture(kOutputNRDSpecularReflectance);
    var["deltaReflectionReflectance"] = renderData.getTexture(kOutputNRDDeltaReflectionReflectance);
    var["deltaReflectionEmission"] = renderData.getTexture(kOutputNRDDeltaReflectionEmission);
    var["deltaReflectionNormWRoughMaterialID"] = renderData.getTexture(kOutputNRDDeltaReflectionNormWRoughMaterialID);
    var["deltaReflectionPathLength"] = renderData.getTexture(kOutputNRDDeltaReflectionPathLength);
    var["deltaReflectionHitDist"] = renderData.getTexture(kOutputNRDDeltaReflectionHitDist);
    var["deltaTransmissionReflectance"] = renderData.getTexture(kOutputNRDDeltaTransmissionReflectance);
    var["deltaTransmissionEmission"] = renderData.getTexture(kOutputNRDDeltaTransmissionEmission);
    var["deltaTransmissionNormWRoughMaterialID"] = renderData.getTexture(kOutputNRDDeltaTransmissionNormWRoughMaterialID);
    var["deltaTransmissionPathLength"] = renderData.getTexture(kOutputNRDDeltaTransmissionPathLength);
    var["deltaTransmissionPosW"] = renderData.getTexture(kOutputNRDDeltaTransmissionPosW);
}
```
- 這段程式碼用於設置非重複數據 (NRD) 相關的資源，將它們綁定到給定的葉狀變數 var 中。

- 具體來說：

    - var["sampleRadiance"] 到 var["sampleReflectance"] 這些葉狀變數將對應到非重複數據的不同部分，如輻射、命中距離、發射等。
    - var["primaryHitEmission"] 到 var["deltaTransmissionPosW"] 這些葉狀變數將對應到渲染數據中與非重複數據相關的紋理，如主要命中發射、主要命中漫反射反射率、主要命中的粗糙度材料ID 等。

- 這些紋理和資源將在後續的光線追蹤過程中使用，用於計算和渲染非重複數據相關的效果。

```cpp
void PathTracer::setShaderData(const ShaderVar& var, const RenderData& renderData, bool useLightSampling) const
{
    // Bind static resources that don't change per frame.
    if (mVarsChanged)
    {
        if (useLightSampling && mpEnvMapSampler) mpEnvMapSampler->setShaderData(var["envMapSampler"]);

        var["sampleOffset"] = mpSampleOffset; // Can be nullptr
        var["sampleGuideData"] = mpSampleGuideData;
    }

    // Bind runtime data.
    setNRDData(var["outputNRD"], renderData);

    Texture::SharedPtr pViewDir;
    if (mpScene->getCamera()->getApertureRadius() > 0.f)
    {
        pViewDir = renderData.getTexture(kInputViewDir);
        if (!pViewDir) logWarning("Depth-of-field requires the '{}' input. Expect incorrect rendering.", kInputViewDir);
    }

    Texture::SharedPtr pSampleCount;
    if (!mFixedSampleCount)
    {
        pSampleCount = renderData.getTexture(kInputSampleCount);
        if (!pSampleCount) throw RuntimeError("PathTracer: Missing sample count input texture");
    }

    var["params"].setBlob(mParams);
    var["vbuffer"] = renderData.getTexture(kInputVBuffer);
    var["viewDir"] = pViewDir; // Can be nullptr
    var["sampleCount"] = pSampleCount; // Can be nullptr
    var["outputColor"] = renderData.getTexture(kOutputColor);

    if (useLightSampling && mpEmissiveSampler)
    {
        // TODO: Do we have to bind this every frame?
        mpEmissiveSampler->setShaderData(var["emissiveSampler"]);
    }
    if (mParams.useConditionalReSTIR) mpConditionalReSTIRPass->setShaderData(var["restir"]);
}
```
- 這段程式碼用於設置光線追蹤的葉狀變數數據，將它們綁定到給定的葉狀變數 var 中。這些資源在光線追蹤過程中將用於計算和渲染。

- 具體來說：

    - 如果 mVarsChanged 發生變化，表示靜態資源需要重新綁定，例如環境貼圖采樣器和樣本偏移資料。這些資源將綁定到 var 的相應葉狀變數中。
    - setNRDData(var["outputNRD"], renderData) 用於設置非重複數據相關的資源。
    - var["params"].setBlob(mParams) 將光線追蹤的參數設置為結構體 mParams。
    - var["vbuffer"] 用於綁定視圖緩衝區的紋理。
    - var["viewDir"] 和 var["sampleCount"] 用於綁定視圖方向和樣本計數的紋理，這些紋理將在深度場景中使用。
    - var["outputColor"] 用於綁定輸出顏色的紋理。
    - 如果使用光線采樣並且存在發射采樣器 mpEmissiveSampler，則將其資源綁定到 var["emissiveSampler"] 中。
    - 如果使用條件性 ReSTIR (mParams.useConditionalReSTIR)，則將相關資源綁定到 var["restir"] 中。

- 這些資源和數據將在每一幀的光線追蹤運算中使用，用於準確計算和渲染場景的光線追蹤效果。

```cpp
bool PathTracer::beginFrame(RenderContext* pRenderContext, const RenderData& renderData)
{
    // Update the random seed.
    mParams.seed = mParams.useFixedSeed ? mParams.fixedSeed : mParams.frameCount + mSeedOffset;

    if (mpConditionalReSTIRPass) mParams.specularRoughnessThreshold = mpConditionalReSTIRPass->getOptions().shiftMappingSettings.specularRoughnessThreshold;

    const auto& pOutputColor = renderData.getTexture(kOutputColor);
    FALCOR_ASSERT(pOutputColor);

    // Set output frame dimension.
    setFrameDim(uint2(pOutputColor->getWidth(), pOutputColor->getHeight()));

    // Validate all I/O sizes match the expected size.
    // If not, we'll disable the path tracer to give the user a chance to fix the configuration before re-enabling it.
    bool resolutionMismatch = false;
    auto validateChannels = [&](const auto& channels) {
        for (const auto& channel : channels)
        {
            auto pTexture = renderData.getTexture(channel.name);
            if (pTexture && (pTexture->getWidth() != mParams.frameDim.x || pTexture->getHeight() != mParams.frameDim.y)) resolutionMismatch = true;
        }
    };
    validateChannels(kInputChannels);
    validateChannels(kOutputChannels);

    if (mEnabled && resolutionMismatch)
    {
        logError("PathTracer I/O sizes don't match. The pass will be disabled.");
        mEnabled = false;
    }

    if (mpScene == nullptr || !mEnabled)
    {
        pRenderContext->clearUAV(pOutputColor->getUAV().get(), float4(0.f));

        // Set refresh flag if changes that affect the output have occured.
        // This is needed to ensure other passes get notified when the path tracer is enabled/disabled.
        if (mOptionsChanged)
        {
            auto& dict = renderData.getDictionary();
            auto flags = dict.getValue(kRenderPassRefreshFlags, Falcor::RenderPassRefreshFlags::None);
            if (mOptionsChanged) flags |= Falcor::RenderPassRefreshFlags::RenderOptionsChanged;
            dict[Falcor::kRenderPassRefreshFlags] = flags;
        }

        return false;
    }

    // Update materials.
    prepareMaterials(pRenderContext);

    // Update the env map and emissive sampler to the current frame.
    bool lightingChanged = prepareLighting(pRenderContext);

    // Prepare RTXDI.
    prepareRTXDI(pRenderContext);
    if (mpRTXDI) mpRTXDI->beginFrame(pRenderContext, mParams.frameDim);

    // Prepare ReSTIR
    if (mParams.useConditionalReSTIR)
    {
        if (!mpConditionalReSTIRPass)
        {
            auto defines = mStaticParams.getDefines(*this); // set owner defines
            mpConditionalReSTIRPass = ConditionalReSTIRPass::create(mpScene, defines, mConditionalReSTIROptions, mpPixelStats);
        }
    }

    if (mpConditionalReSTIRPass)
    {
        mpConditionalReSTIRPass->setPathTracerParams(mParams.useFixedSeed, mParams.fixedSeed,
            mParams.lodBias, mParams.specularRoughnessThreshold, mParams.frameDim, mParams.screenTiles, mParams.frameCount,
            mParams.seed, mParams.samplesPerPixel, mParams.DIMode);
        mpConditionalReSTIRPass->setSharedStaticParams(0, mStaticParams.maxSurfaceBounces, mStaticParams.useNEE);
        mpConditionalReSTIRPass->beginFrame(pRenderContext, mParams.frameDim, mParams.screenTiles, mRecompile);
    }

    // Update refresh flag if changes that affect the output have occured.
    auto& dict = renderData.getDictionary();
    if (mOptionsChanged || lightingChanged)
    {
        auto flags = dict.getValue(kRenderPassRefreshFlags, Falcor::RenderPassRefreshFlags::None);
        if (mOptionsChanged) flags |= Falcor::RenderPassRefreshFlags::RenderOptionsChanged;
        if (lightingChanged) flags |= Falcor::RenderPassRefreshFlags::LightingChanged;
        dict[Falcor::kRenderPassRefreshFlags] = flags;
        mOptionsChanged = false;
    }

    // Check if GBuffer has adjusted shading normals enabled.
    bool gbufferAdjustShadingNormals = dict.getValue(Falcor::kRenderPassGBufferAdjustShadingNormals, false);
    if (gbufferAdjustShadingNormals != mGBufferAdjustShadingNormals)
    {
        mGBufferAdjustShadingNormals = gbufferAdjustShadingNormals;
        mRecompile = true;
    }

    // Check if fixed sample count should be used. When the sample count input is connected we load the count from there instead.
    mFixedSampleCount = renderData[kInputSampleCount] == nullptr;

    // Check if guide data should be generated.
    mOutputGuideData = renderData[kOutputAlbedo] != nullptr || renderData[kOutputSpecularAlbedo] != nullptr
        || renderData[kOutputIndirectAlbedo] != nullptr || renderData[kOutputNormal] != nullptr
        || renderData[kOutputReflectionPosW] != nullptr;

    // Check if NRD data should be generated.
    mOutputNRDData =
        renderData[kOutputNRDDiffuseRadianceHitDist] != nullptr
        || renderData[kOutputNRDSpecularRadianceHitDist] != nullptr
        || renderData[kOutputNRDResidualRadianceHitDist] != nullptr
        || renderData[kOutputNRDEmission] != nullptr
        || renderData[kOutputNRDDiffuseReflectance] != nullptr
        || renderData[kOutputNRDSpecularReflectance] != nullptr;

    // Check if additional NRD data should be generated.
    bool prevOutputNRDAdditionalData = mOutputNRDAdditionalData;
    mOutputNRDAdditionalData =
        renderData[kOutputNRDDeltaReflectionRadianceHitDist] != nullptr
        || renderData[kOutputNRDDeltaTransmissionRadianceHitDist] != nullptr
        || renderData[kOutputNRDDeltaReflectionReflectance] != nullptr
        || renderData[kOutputNRDDeltaReflectionEmission] != nullptr
        || renderData[kOutputNRDDeltaReflectionNormWRoughMaterialID] != nullptr
        || renderData[kOutputNRDDeltaReflectionPathLength] != nullptr
        || renderData[kOutputNRDDeltaReflectionHitDist] != nullptr
        || renderData[kOutputNRDDeltaTransmissionReflectance] != nullptr
        || renderData[kOutputNRDDeltaTransmissionEmission] != nullptr
        || renderData[kOutputNRDDeltaTransmissionNormWRoughMaterialID] != nullptr
        || renderData[kOutputNRDDeltaTransmissionPathLength] != nullptr
        || renderData[kOutputNRDDeltaTransmissionPosW] != nullptr;
    if (mOutputNRDAdditionalData != prevOutputNRDAdditionalData) mRecompile = true;

    // Enable pixel stats if rayCount or pathLength outputs are connected.
    if (renderData[kOutputRayCount] != nullptr || renderData[kOutputPathLength] != nullptr)
    {
        mpPixelStats->setEnabled(true);
    }

    mpPixelStats->beginFrame(pRenderContext, mParams.frameDim);
    mpPixelDebug->beginFrame(pRenderContext, mParams.frameDim);

    if (!mpSavedOutput)
    {
        int frameDimX = renderData.getTexture(kOutputColor)->getWidth();
        int frameDimY = renderData.getTexture(kOutputColor)->getHeight();
        mpSavedOutput = Texture::create2D(frameDimX, frameDimY, renderData.getTexture(kOutputColor)->getFormat(), 1, 1, nullptr, ResourceBindFlags::ShaderResource | ResourceBindFlags::UnorderedAccess);
    }

    return true;
}
```
- 這段程式碼是 PathTracer 類中的 beginFrame 方法，用於準備新的渲染幀。它執行了以下操作：

    1. 更新隨機種子 (mParams.seed)。如果使用固定種子 (mParams.useFixedSeed)，則使用固定種子值 (mParams.fixedSeed)；否則，使用每幀的計數加上偏移量 (mSeedOffset) 作為隨機種子。
    2. 確認輸出顏色紋理 (pOutputColor) 存在並獲取其維度，並設置到 PathTracer 中。
    3. 驗證所有的輸入和輸出紋理與期望的大小是否匹配。如果不匹配，禁用 PathTracer，並記錄錯誤信息。
    4. 如果 PathTracer 無效或場景為空，則清除輸出顏色紋理並返回 false。
    5. 如果有材質變化或照明變化，設置相應的刷新標誌，以通知其他通道進行更新。
    6. 準備材質。
    7. 更新照明設置，並檢查是否發生了照明變化。
    8. 準備 RTXDI，如果啟用了 RTXDI 的使用。
    9. 如果使用條件性 ReSTIR，則準備相關設置。
    10. 檢查並設置渲染選項的刷新標誌。
    11. 檢查是否啟用了 GBuffer 的調整照明法向量。
    12. 檢查是否應生成引導數據 (mOutputGuideData) 和 NRD 數據 (mOutputNRDData)。
    13. 如果已連接了輸出的額外 NRD 數據，則檢查是否應重新編譯。
    14. 如果輸出了射線計數或路徑長度，則啟用像素統計。
    15. 開始像素統計和像素調試。
    16. 創建一個紋理 (mpSavedOutput) 來保存輸出。

- 最終，返回 true，表示成功準備了新的渲染幀。

```cpp
void PathTracer::endFrame(RenderContext* pRenderContext, const RenderData& renderData)
{
    mpPixelStats->endFrame(pRenderContext);
    mpPixelDebug->endFrame(pRenderContext);

    auto copyTexture = [pRenderContext](Texture* pDst, const Texture* pSrc)
    {
        if (pDst && pSrc)
        {
            FALCOR_ASSERT(pDst && pSrc);
            FALCOR_ASSERT(pDst->getFormat() == pSrc->getFormat());
            FALCOR_ASSERT(pDst->getWidth() == pSrc->getWidth() && pDst->getHeight() == pSrc->getHeight());
            pRenderContext->copyResource(pDst, pSrc);
        }
        else if (pDst)
        {
            pRenderContext->clearUAV(pDst->getUAV().get(), uint4(0, 0, 0, 0));
        }
    };

    // Copy pixel stats to outputs if available.
    copyTexture(renderData.getTexture(kOutputRayCount).get(), mpPixelStats->getRayCountTexture(pRenderContext).get());
    copyTexture(renderData.getTexture(kOutputPathLength).get(), mpPixelStats->getPathLengthTexture().get());

    if (mpRTXDI) mpRTXDI->endFrame(pRenderContext);
    if (mpConditionalReSTIRPass) mpConditionalReSTIRPass->endFrame(pRenderContext);

    mVarsChanged = false;
    if (!mpScene->freeze) mParams.frameCount++;
}
```
- 這段程式碼是 PathTracer 類中的 endFrame 方法，用於完成當前渲染幀的結束操作。它執行了以下操作：

    1. 結束像素統計和像素調試。
    2. 定義了一個內部函數 copyTexture 來從像素統計數據中複製數據到輸出紋理。這個函數檢查輸入的目標紋理和源紋理是否存在，並且它們的格式和大小是否匹配，然後進行相應的複製操作或清空操作。
    3. 將像素統計的射線計數和路徑長度複製到相應的輸出紋理中（如果可用）。
    4. 如果啟用了 RTXDI，則結束 RTXDI 的當前渲染幀。
    5. 如果使用了條件性 ReSTIR，則結束 ReSTIR 的當前渲染幀。
    6. 將變數變化標誌 (mVarsChanged) 設置為 false。
    7. 如果場景沒有被凍結，則增加渲染幀計數 (mParams.frameCount)。

- 這個方法的主要目的是進行渲染幀的後處理操作，包括統計數據的複製和計數器的增加。

```cpp
void PathTracer::generatePaths(RenderContext* pRenderContext, const RenderData& renderData)
{
    FALCOR_PROFILE("generatePaths");

    // Check shader assumptions.
    // We launch one thread group per screen tile, with threads linearly indexed.
    const uint32_t tileSize = kScreenTileDim.x * kScreenTileDim.y;
    FALCOR_ASSERT(kScreenTileDim.x == 16 && kScreenTileDim.y == 16); // TODO: Remove this temporary limitation when Slang bug has been fixed, see comments in shader.
    FALCOR_ASSERT(kScreenTileBits.x <= 4 && kScreenTileBits.y <= 4); // Since we use 8-bit deinterleave.
    FALCOR_ASSERT(mpGeneratePaths->getThreadGroupSize().x == tileSize);
    FALCOR_ASSERT(mpGeneratePaths->getThreadGroupSize().y == 1 && mpGeneratePaths->getThreadGroupSize().z == 1);

    // Additional specialization. This shouldn't change resource declarations.
    mpGeneratePaths->addDefine("USE_VIEW_DIR", (mpScene->getCamera()->getApertureRadius() > 0 && renderData[kInputViewDir] != nullptr) ? "1" : "0");
    mpGeneratePaths->addDefine("OUTPUT_GUIDE_DATA", mOutputGuideData ? "1" : "0");
    mpGeneratePaths->addDefine("OUTPUT_NRD_DATA", mOutputNRDData ? "1" : "0");
    mpGeneratePaths->addDefine("OUTPUT_NRD_ADDITIONAL_DATA", mOutputNRDAdditionalData ? "1" : "0");

    // Bind resources.
    auto var = mpGeneratePaths->getRootVar()["CB"]["gPathGenerator"];
    setShaderData(var, renderData, false);
    var["resetTemporal"] = mpConditionalReSTIRPass && mpConditionalReSTIRPass->needResetTemporalHistory();

    mpGeneratePaths["gScene"] = mpScene->getParameterBlock();

    if (mpRTXDI) mpRTXDI->setShaderData(mpGeneratePaths->getRootVar());

    // Launch one thread per pixel.
    // The dimensions are padded to whole tiles to allow re-indexing the threads in the shader.
    mpGeneratePaths->execute(pRenderContext, { mParams.screenTiles.x * tileSize, mParams.screenTiles.y, 1u });
}
```
- 這段程式碼是 PathTracer 類中的 generatePaths 方法，用於生成光線路徑。該方法執行以下操作：

    1. 使用 FALCOR_PROFILE 宏將該方法標記為 "generatePaths"，以便於性能分析。
    2. 檢查 shader 的假設前提。該方法假設每個屏幕瓦片啟動一個線程組，並且每個線程按線性索引排列。
    3. 對生成路徑的 ComputePass 進行進一步的特化。這些特化不應該改變資源聲明。
    4. 綁定資源，包括相機、輸入數據等。
    5. 如果需要重置暫時歷史，設置 resetTemporal 變量。
    6. 將場景的參數綁定到 ComputePass 中。
    7. 如果啟用了 RTXDI，將其 shader 數據綁定到 ComputePass 中。
    8. 使用 execute 方法執行 ComputePass，以在每個像素上生成路徑。計算執行的維度為 (mParams.screenTiles.x * tileSize, mParams.screenTiles.y, 1u)，其中 tileSize 是屏幕瓦片的大小，即 kScreenTileDim.x * kScreenTileDim.y。

- 總的來說，這個方法是 PathTracer 類中的核心方法之一，負責在每個渲染幀中生成光線路徑。

```cpp
void PathTracer::tracePass(RenderContext* pRenderContext, const RenderData& renderData, const ComputePass::SharedPtr& pTracePass, uint32_t curIter)
{
    FALCOR_PROFILE("Trace Pass");

    // Additional specialization. This shouldn't change resource declarations.
    pTracePass->addDefine("USE_VIEW_DIR", (mpScene->getCamera()->getApertureRadius() > 0 && renderData[kInputViewDir] != nullptr) ? "1" : "0");
    pTracePass->addDefine("OUTPUT_GUIDE_DATA", mOutputGuideData ? "1" : "0");
    pTracePass->addDefine("OUTPUT_NRD_DATA", mOutputNRDData ? "1" : "0");
    pTracePass->addDefine("OUTPUT_NRD_ADDITIONAL_DATA", mOutputNRDAdditionalData ? "1" : "0");

    // Bind global resources.
    auto var = pTracePass->getRootVar();
    mpScene->setRaytracingShaderData(pRenderContext, var);

    if (mVarsChanged) mpSampleGenerator->setShaderData(var);
    if (mpRTXDI) mpRTXDI->setShaderData(var);

    mpPixelStats->prepareProgram(pTracePass->getProgram(), var);
    mpPixelDebug->prepareProgram(pTracePass->getProgram(), var);

    // Bind the path tracer.
    var["gPathTracer"] = mpPathTracerBlock;
    var["gScheduler"]["curIter"] = curIter;

    // Full screen dispatch.
    pTracePass->execute(pRenderContext, uint3(mParams.frameDim, 1u));
}
```
- 這段程式碼是 PathTracer 類中的 tracePass 方法，用於執行光線追蹤的過程。

    1. 使用 FALCOR_PROFILE 宏將該方法標記為 "Trace Pass"，以便於性能分析。
    2. 對光線追蹤的 ComputePass 進行進一步的特化。這些特化不應該改變資源聲明。
    3. 綁定全局資源，包括場景的光線追蹤數據和其他相關資源。
    4. 如果變量已更改，將樣本生成器的 shader 數據綁定到 ComputePass 中。
    5. 如果啟用了 RTXDI，將其 shader 數據綁定到 ComputePass 中。
    6. 使用像素統計和像素調試功能，準備程序，以便它們可以在光線追蹤中使用。
    7. 綁定路徑追蹤器（PathTracer）參數块。
    8. 使用全屏分派（full screen dispatch）的方法執行 ComputePass。計算執行的維度為 (mParams.frameDim, 1u)，即每個像素一個執行緒組。

- 總的來說，這個方法是 PathTracer 類中負責執行光線追蹤的核心方法之一。

```cpp
void PathTracer::resolvePass(RenderContext* pRenderContext, const RenderData& renderData)
{
    if (!mOutputGuideData && !mOutputNRDData && !mParams.useConditionalReSTIR && mFixedSampleCount && mParams.samplesPerPixel == 1) return;

    FALCOR_PROFILE("resolvePass");

    // This pass is executed when multiple samples per pixel are used.
    // We launch one thread per pixel that computes the resolved color by iterating over the samples.
    // The samples are arranged in tiles with pixels in Morton order, with samples stored consecutively for each pixel.
    // With adaptive sampling, an extra sample offset lookup table computed by the path generation pass is used to
    // locate the samples for each pixel.

    // Additional specialization. This shouldn't change resource declarations.
    mpResolvePass->addDefine("OUTPUT_GUIDE_DATA", mOutputGuideData ? "1" : "0");
    mpResolvePass->addDefine("OUTPUT_NRD_DATA", mOutputNRDData ? "1" : "0");

    // Bind resources.
    auto var = mpResolvePass->getRootVar()["CB"]["gResolvePass"];
    var["params"].setBlob(mParams);
    var["sampleCount"] = renderData.getTexture(kInputSampleCount); // Can be nullptr
    var["outputColor"] = renderData.getTexture(kOutputColor);
    var["outputAlbedo"] = renderData.getTexture(kOutputAlbedo);
    var["outputSpecularAlbedo"] = renderData.getTexture(kOutputSpecularAlbedo);
    var["outputIndirectAlbedo"] = renderData.getTexture(kOutputIndirectAlbedo);
    var["outputNormal"] = renderData.getTexture(kOutputNormal);
    var["outputReflectionPosW"] = renderData.getTexture(kOutputReflectionPosW);
    var["outputNRDDiffuseRadianceHitDist"] = renderData.getTexture(kOutputNRDDiffuseRadianceHitDist);
    var["outputNRDSpecularRadianceHitDist"] = renderData.getTexture(kOutputNRDSpecularRadianceHitDist);
    var["outputNRDDeltaReflectionRadianceHitDist"] = renderData.getTexture(kOutputNRDDeltaReflectionRadianceHitDist);
    var["outputNRDDeltaTransmissionRadianceHitDist"] = renderData.getTexture(kOutputNRDDeltaTransmissionRadianceHitDist);
    var["outputNRDResidualRadianceHitDist"] = renderData.getTexture(kOutputNRDResidualRadianceHitDist);
    var["vbuffer"] = renderData.getTexture(kInputVBuffer);
    if (mpConditionalReSTIRPass)
    {
        mpConditionalReSTIRPass->setReservoirData(var);
    }

    if (mVarsChanged)
    {
        var["sampleOffset"] = mpSampleOffset; // Can be nullptr
        var["sampleGuideData"] = mpSampleGuideData;
        var["sampleNRDRadiance"] = mpSampleNRDRadiance;
        var["sampleNRDHitDist"] = mpSampleNRDHitDist;
        var["sampleNRDEmission"] = mpSampleNRDEmission;
        var["sampleNRDReflectance"] = mpSampleNRDReflectance;

        var["sampleNRDPrimaryHitNeeOnDelta"] = mpSampleNRDPrimaryHitNeeOnDelta;
        var["primaryHitDiffuseReflectance"] = renderData.getTexture(kOutputNRDDiffuseReflectance);
    }
    var["outputColorPrev"] = mpSavedOutput;
    var["freeze"] = mpScene->freeze && mIsFrozen;

    // Launch one thread per pixel.
    mpResolvePass->execute(pRenderContext, { mParams.frameDim, 1u });
}
```
- 這段程式碼是 PathTracer 類中的 resolvePass 方法，用於解析多個每像素的樣本。

    1. 首先，檢查是否需要執行解析過程。如果沒有輔助數據、無非重要數據、不使用條件式 ReSTIR、固定每像素樣本數為 1，則直接返回，不執行解析過程。
    2. 使用 FALCOR_PROFILE 宏將該方法標記為 "resolvePass"，以便於性能分析。
    3. 添加額外的特化。這些特化不應該改變資源聲明。
    4. 綁定資源。這些資源包括場景的相關資料、輸入和輸出的紋理等。
    5. 如果變量已更改，綁定樣本偏移、樣本引導數據、樣本非重要數據的輻射、擊中距離、發射和反射率等資源。
    6. 設置先前的輸出顏色紋理為 mpSavedOutput。
    7. 設置凍結標誌為 mpScene->freeze 和 mIsFrozen 的邏輯與。

- 最後，使用單線程解析每個像素。計算執行的維度為 (mParams.frameDim, 1u)。

```cpp
void PathTracer::setPathTracerDataForConditionalReSTIR(const ShaderVar& var, const RenderData& renderData, bool useLightSampling) const
{
    // Bind static resources that don't change per frame.
    if (mVarsChanged)
    {
        if (useLightSampling && mpEnvMapSampler) mpEnvMapSampler->setShaderData(var["envMapSampler"]);
    }

    Texture::SharedPtr pViewDir;
    if (mpScene->getCamera()->getApertureRadius() > 0.f)
    {
        pViewDir = renderData.getTexture(kInputViewDir);
        if (!pViewDir) logWarning("Depth-of-field requires the '{}' input. Expect incorrect rendering.", kInputViewDir);
    }

    Texture::SharedPtr pSampleCount;
    if (!mFixedSampleCount)
    {
        pSampleCount = renderData.getTexture(kInputSampleCount);
        if (!pSampleCount) throw RuntimeError("PathTracer: Missing sample count input texture");
    }

    var["params"].setBlob(mParams);
    var["vbuffer"] = renderData.getTexture(kInputVBuffer);
    var["viewDir"] = pViewDir; // Can be nullptr
    var["sampleCount"] = pSampleCount; // Can be nullptr
    if (useLightSampling && mpEmissiveSampler)
    {
        // TODO: Do we have to bind this every frame?
        mpEmissiveSampler->setShaderData(var["emissiveSampler"]);
    }
    var["outputColor"] = renderData.getTexture(kOutputColor);

    if (mParams.useConditionalReSTIR) mpConditionalReSTIRPass->setShaderData(var["restir"]);
}
```
- 這是 PathTracer 類中的 setPathTracerDataForConditionalReSTIR 方法。這個方法用於為條件式 ReSTIR 算法設置路徑追蹤器數據。

    1. 首先，檢查是否需要綁定靜態資源，即不會隨每幀變化的資源。如果 mVarsChanged 為真，則執行以下操作：
        - 如果 useLightSampling 為真且 mpEnvMapSampler 存在，則為 envMapSampler 綁定着色器數據。
    2. 如果相機的光圈半徑大於 0，則獲取輸入的視角方向紋理，如果不存在，則記錄警告。
    3. 如果不是固定的樣本數，則獲取輸入的樣本計數紋理，如果不存在，則拋出運行時錯誤。
    4. 設置變量的 blob 數據為 mParams，以及相應的紋理變量，如視角方向、樣本計數和輸出顏色。
    5. 如果使用光線採樣且 mpEmissiveSampler 存在，則為 emissiveSampler 綁定着色器數據。
    6. 如果啟用了條件式 ReSTIR，則為 restir 綁定着色器數據。

- 最後，通過將參數 var 傳遞給函數的調用者，將所需的着色器數據傳遞給條件式 ReSTIR 算法。

```cpp
void PathTracer::setPresetForMethod(int id, bool fromGui)
{
    mpConditionalReSTIRPass->mReallocate = true;
    mpConditionalReSTIRPass->mRecompile = true;
    mpConditionalReSTIRPass->mResetTemporalReservoirs = true;
    mRecompile = true;
    mOptionsChanged = true;

    mSavedPTSpp[mPrevRenderModePresetId] = mParams.samplesPerPixel;
    mPrevRenderModePresetId = id;

    if (id == 0) // Conditional ReSTIR
    {
        mParams.samplesPerPixel = 1;
        mpConditionalReSTIRPass->getOptions().subpathSetting.useMMIS = false;
        mpConditionalReSTIRPass->getOptions().shiftMapping = ConditionalReSTIR::ShiftMapping::Hybrid;
        mpConditionalReSTIRPass->getOptions().subpathSetting.suffixSpatialReuseRounds = 1;
        mpConditionalReSTIRPass->getOptions().subpathSetting.temporalHistoryLength = 50;
        mpConditionalReSTIRPass->getOptions().subpathSetting.suffixTemporalReuse = true;
        mParams.useConditionalReSTIR = true;
    }
    else if (id == 1) // MMIS
    {
        mParams.samplesPerPixel = 1;
        mpConditionalReSTIRPass->getOptions().subpathSetting.useMMIS = true;
        mpConditionalReSTIRPass->getOptions().shiftMapping = ConditionalReSTIR::ShiftMapping::Reconnection;
        mpConditionalReSTIRPass->getOptions().subpathSetting.suffixSpatialReuseRounds = 0;
        mpConditionalReSTIRPass->getOptions().subpathSetting.temporalHistoryLength = 0;
        mpConditionalReSTIRPass->getOptions().subpathSetting.suffixTemporalReuse = false;
        mParams.useConditionalReSTIR = true;
    }
    else if (id == 2) // path tracing
    {
        if (fromGui) mParams.samplesPerPixel = mSavedPTSpp[2];
        mpConditionalReSTIRPass->getOptions().subpathSetting.useMMIS = false;
        mpConditionalReSTIRPass->getOptions().shiftMapping = ConditionalReSTIR::ShiftMapping::Hybrid;
        mpConditionalReSTIRPass->getOptions().subpathSetting.suffixSpatialReuseRounds = 1;
        mpConditionalReSTIRPass->getOptions().subpathSetting.temporalHistoryLength = 50;
        mpConditionalReSTIRPass->getOptions().subpathSetting.suffixTemporalReuse = true;
        mParams.useConditionalReSTIR = false;
    }
}
```
- 這是 PathTracer 類中的 setPresetForMethod 方法。這個方法用於根據給定的預設值 ID 設置路徑追蹤器的參數。

    1. 將條件式 ReSTIR 算法的一些標誌設置為真，以便在下一次渲染時重新分配、重新編譯和重置臨時水庫。
    2. 將 mRecompile 和 mOptionsChanged 設置為真，以確保重新編譯程序和更新選項。
    3. 將當前渲染模式預設 ID 存儲在 mPrevRenderModePresetId 中，並將上一個渲染模式的樣本每像素數量保存在 mSavedPTSpp 中。
    4. 根據給定的 ID 選擇不同的預設值：
        - 如果 ID 為 0，表示使用條件式 ReSTIR 算法，將樣本每像素數量設置為 1，並設置條件式 ReSTIR 算法的相關選項。
        - 如果 ID 為 1，表示使用 MMIS 算法，同樣將樣本每像素數量設置為 1，並設置 MMIS 算法的相關選項。
        - 如果 ID 為 2，表示使用路徑追蹤，根據需要從 GUI 中設置樣本每像素數量，並將條件式 ReSTIR 算法的相關選項設置為路徑追蹤模式。

- 總之，這個方法根據給定的預設值 ID 選擇不同的路徑追蹤算法或模式，並相應地設置相關的參數。

```cpp
Program::DefineList PathTracer::StaticParams::getDefines(const PathTracer& owner) const
{
    Program::DefineList defines;

    // Path tracer configuration.
    defines.add("MAX_SURFACE_BOUNCES", std::to_string(maxSurfaceBounces));
    defines.add("MAX_DIFFUSE_BOUNCES", std::to_string(maxDiffuseBounces));
    defines.add("MAX_SPECULAR_BOUNCES", std::to_string(maxSpecularBounces));
    defines.add("MAX_TRANSMISSON_BOUNCES", std::to_string(maxTransmissionBounces));
    defines.add("ADJUST_SHADING_NORMALS", adjustShadingNormals ? "1" : "0");
    defines.add("USE_BSDF_SAMPLING", useBSDFSampling ? "1" : "0");
    defines.add("USE_NEE", useNEE ? "1" : "0");
    defines.add("USE_MIS", useMIS && useNEE ? "1" : "0");
    defines.add("USE_RUSSIAN_ROULETTE", useRussianRoulette ? "1" : "0");
    defines.add("USE_RTXDI", useRTXDI ? "1" : "0");
    defines.add("USE_ALPHA_TEST", useAlphaTest ? "1" : "0");
    defines.add("USE_LIGHTS_IN_DIELECTRIC_VOLUMES", useLightsInDielectricVolumes ? "1" : "0");
    defines.add("DISABLE_CAUSTICS", disableCaustics ? "1" : "0");
    defines.add("PRIMARY_LOD_MODE", std::to_string((uint32_t)primaryLodMode));
    defines.add("USE_NRD_DEMODULATION", useNRDDemodulation ? "1" : "0");
    defines.add("COLOR_FORMAT", std::to_string((uint32_t)colorFormat));
    defines.add("MIS_HEURISTIC", std::to_string((uint32_t)misHeuristic));
    defines.add("MIS_POWER_EXPONENT", std::to_string(misPowerExponent));

    // Sampling utilities configuration.
    FALCOR_ASSERT(owner.mpSampleGenerator);
    defines.add(owner.mpSampleGenerator->getDefines());

    if (owner.mpEmissiveSampler) defines.add(owner.mpEmissiveSampler->getDefines());
    if (owner.mpRTXDI) defines.add(owner.mpRTXDI->getDefines());

    defines.add("INTERIOR_LIST_SLOT_COUNT", std::to_string(maxNestedMaterials));

    defines.add("GBUFFER_ADJUST_SHADING_NORMALS", owner.mGBufferAdjustShadingNormals ? "1" : "0");

    // Scene-specific configuration.
    const auto& scene = owner.mpScene;
    if (scene) defines.add(scene->getSceneDefines());
    defines.add("USE_ENV_LIGHT", scene && scene->useEnvLight() ? "1" : "0");
    defines.add("USE_ANALYTIC_LIGHTS", scene && scene->useAnalyticLights() ? "1" : "0");
    defines.add("USE_EMISSIVE_LIGHTS", scene && scene->useEmissiveLights() ? "1" : "0");
    defines.add("USE_CURVES", scene && (scene->hasGeometryType(Scene::GeometryType::Curve)) ? "1" : "0");
    defines.add("USE_SDF_GRIDS", scene && scene->hasGeometryType(Scene::GeometryType::SDFGrid) ? "1" : "0");
    defines.add("USE_HAIR_MATERIAL", scene && scene->getMaterialCountByType(MaterialType::Hair) > 0u ? "1" : "0");

    // Set default (off) values for additional features.
    defines.add("USE_VIEW_DIR", "0");
    defines.add("OUTPUT_GUIDE_DATA", "0");
    defines.add("OUTPUT_NRD_DATA", "0");
    defines.add("OUTPUT_NRD_ADDITIONAL_DATA", "0");

    defines.add("DiffuseBrdf", useLambertianDiffuse ? "DiffuseBrdfLambert" : "DiffuseBrdfFrostbite");
    defines.add("enableDiffuse", disableDiffuse ? "0" : "1");
    defines.add("enableSpecular", disableSpecular ? "0" : "1");
    defines.add("enableTranslucency", disableTranslucency ? "0" : "1");

    if (owner.mpConditionalReSTIRPass)
    {
        owner.mpConditionalReSTIRPass->setOwnerDefines(defines);
        defines.add(owner.mpConditionalReSTIRPass->getDefines());
    }

    return defines;
}
```
- 這是 PathTracer 類中的 StaticParams::getDefines 方法。這個方法用於根據路徑追蹤器的靜態參數和相關的渲染器配置，生成用於編譯着色器程序的預處理器定義列表。
- 這些定義包括了路徑追蹤器的配置、采樣工具的配置、場景特定的配置，以及其他附加功能的配置。該方法首先創建一個空的預處理器定義列表，然後根據路徑追蹤器的參數設置各種預處理器定義，最終返回生成的定義列表。
- 這樣的定義列表用於在編譯着色器程序時，根據不同的配置生成不同的着色器代碼，從而實現路徑追蹤器的各種功能和效果。

- 每一項的意思：

    - MAX_SURFACE_BOUNCES: 最大表面反射次數。
    - MAX_DIFFUSE_BOUNCES: 最大漫反射次數。
    - MAX_SPECULAR_BOUNCES: 最大鏡面反射次數。
    - MAX_TRANSMISSION_BOUNCES: 最大折射次數。
    - ADJUST_SHADING_NORMALS: 是否調整漫反射的表面法線。
    - USE_BSDF_SAMPLING: 是否使用 BSDF 采樣。
    - USE_NEE: 是否使用 Next Event Estimation (NEE)。
    - USE_MIS: 是否使用 Multiple Importance Sampling (MIS)。
    - USE_RUSSIAN_ROULETTE: 是否使用俄羅斯輪盤法減少光線數量。
    - USE_RTXDI: 是否使用 RTXDI。
    - USE_ALPHA_TEST: 是否使用 Alpha 測試。
    - USE_LIGHTS_IN_DIELECTRIC_VOLUMES: 是否在折射體積中使用光源。
    - DISABLE_CAUSTICS: 是否禁用焦散效果。
    - PRIMARY_LOD_MODE: 主要 LOD 模式。
    - USE_NRD_DEMODULATION: 是否使用 NRD 解調。
    - COLOR_FORMAT: 顏色格式。
    - MIS_HEURISTIC: MIS 启发式方法。
    - MIS_POWER_EXPONENT: MIS 功率指數。

- 除了路徑追蹤器的配置外，還有一些針對采樣工具、場景特定功能和其他附加功能的配置：

    - INTERIOR_LIST_SLOT_COUNT: 內部列表槽位數量。
    - GBUFFER_ADJUST_SHADING_NORMALS: GBuffer 是否調整漫反射的表面法線。
    - USE_ENV_LIGHT: 是否使用環境光源。
    - USE_ANALYTIC_LIGHTS: 是否使用解析光源。
    - USE_EMISSIVE_LIGHTS: 是否使用發光光源。
    - USE_CURVES: 是否使用曲線。
    - USE_SDF_GRIDS: 是否使用 SDF 网格。
    - USE_HAIR_MATERIAL: 是否使用頭髮材質。

- 此外，還有一些其他功能的配置：

    - USE_VIEW_DIR: 是否使用視線方向。
    - OUTPUT_GUIDE_DATA: 是否輸出引導數據。
    - OUTPUT_NRD_DATA: 是否輸出 NRD 數據。
    - OUTPUT_NRD_ADDITIONAL_DATA: 是否輸出附加的 NRD 數據。
    - DiffuseBrdf: 漫反射 BRDF 的類型。
    - enableDiffuse: 是否啟用漫反射。
    - enableSpecular: 是否啟用鏡面反射。
    - enableTranslucency: 是否啟用半透明效果。

- 這些預處理器定義將在編譯着色器程序時使用，以根據不同的配置生成相應的着色器代碼，從而實現路徑追蹤器的各種功能和效果。
