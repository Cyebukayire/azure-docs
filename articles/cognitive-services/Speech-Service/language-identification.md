---
title: Language identification - Speech service
titleSuffix: Azure Cognitive Services
description: Language identification is used to determine the language being spoken in audio when compared against a list of provided languages.
services: cognitive-services
author: eric-urban
manager: nitinme
ms.service: cognitive-services
ms.subservice: speech-service
ms.topic: how-to
ms.date: 02/13/2022
ms.author: eur
zone_pivot_groups: programming-languages-speech-services-nomore-variant
---

# Language identification (preview)

Language identification is used to identify languages spoken in audio when compared against a list of [supported languages](language-support.md#language-identification).

Language identification (LID) use cases include:

* [Standalone language identification](#standalone-language-identification) when you only need to identify the language in an audio source.
* [Speech-to-text recognition](#speech-to-text) when you need to identify the language in an audio source and then transcribe it to text.
* [Speech translation](#speech-translation) when you need to identify the language in an audio source and then translate it to another language.

Note that for speech recognition, the initial latency is higher with language identification. You should only include this optional feature as needed.

## Configuration options

Whether you use language identification [on its own](#standalone-language-identification), with [speech-to-text](#speech-to-text), or with [speech translation](#speech-translation), there are some common concepts and configuration options.

- Define a list of [candidate languages](#candidate-languages) that you expect in the audio.
- Decide whether to use [at-start or continuous](#at-start-and-continuous-language-identification) language identification.
- Prioritize [low latency or high accuracy](#accuracy-and-latency-prioritization) of results.

Then you make a [recognize once or continuous recognition](#recognize-once-or-continuous) request to the Speech service.

Code snippets are included with the concepts described next. Complete samples for each use case are provided further below.

### Candidate languages

You provide candidate languages, at least one of which is expected be in the audio. You can include up to 4 languages for [at-start LID](#at-start-and-continuous-language-identification) or up to 10 languages for [continuous LID](#at-start-and-continuous-language-identification).

You must provide the full 4-letter locale, but language identification only uses one locale per base language. Do not include multiple locales (e.g., "en-US" and "en-GB") for the same language.

::: zone pivot="programming-language-csharp"

```csharp
var autoDetectSourceLanguageConfig =
    AutoDetectSourceLanguageConfig.FromLanguages(new string[] { "en-US", "de-DE", "zh-CN" });
```

::: zone-end
::: zone pivot="programming-language-cpp"

```cpp
auto autoDetectSourceLanguageConfig = 
    AutoDetectSourceLanguageConfig::FromLanguages({ "en-US", "de-DE", "zh-CN" });
```

::: zone-end
::: zone pivot="programming-language-python"

```python
auto_detect_source_language_config = \
    speechsdk.languageconfig.AutoDetectSourceLanguageConfig(languages=["en-US", "de-DE", "zh-CN"])
```

::: zone-end
::: zone pivot="programming-language-java"

```java
AutoDetectSourceLanguageConfig autoDetectSourceLanguageConfig =
    AutoDetectSourceLanguageConfig.fromLanguages(Arrays.asList("en-US", "de-DE", "zh-CN"));
```

::: zone-end
::: zone pivot="programming-language-javascript"

```javascript
var autoDetectSourceLanguageConfig = SpeechSDK.AutoDetectSourceLanguageConfig.fromLanguages([("en-US", "de-DE", "zh-CN"]);
```

::: zone-end
::: zone pivot="programming-language-objectivec"

```objective-c
NSArray *languages = @[@"en-US", @"de-DE", @"zh-CN"];
SPXAutoDetectSourceLanguageConfiguration* autoDetectSourceLanguageConfig = \
    [[SPXAutoDetectSourceLanguageConfiguration alloc]init:languages];
```

::: zone-end

For more information, see [supported languages](language-support.md#language-identification).

### At-start and Continuous language identification

Speech supports both at-start and continuous language identification (LID).

> [!NOTE]
> Continuous language identification is only supported with Speech SDKs in C#, C++, and Python.
- At-start LID identifies the language once within the first few seconds of audio. Use at-start LID if the language in the audio won't change.
- Continuous LID can identify multiple languages for the duration of the audio. Use continuous LID if the language in the audio could change. Continuous LID does not support changing languages within the same sentence. For example, if you are primarily speaking Spanish and insert some English words, it will not detect the language change per word.

You implement at-start LID or continuous LID by calling methods for [recognize once or continuous](#recognize-once-or-continuous). Results also depend upon your [Accuracy and Latency prioritization](#accuracy-and-latency-prioritization).

### Accuracy and Latency prioritization

You can choose to prioritize accuracy or latency with language identification.

> [!NOTE]
> Latency is prioritized by default with the Speech SDK. You can choose to prioritize accuracy or latency with the Speech SDKs for C#, C++, and Python.
Prioritize `Latency` if you need a low-latency result such as during live streaming. Set the priority to `Accuracy` if the audio quality may be poor, and more latency is acceptable. For example, a voicemail could have background noise, or some silence at the beginning. Allowing the engine more time will improve language identification results.

* **At-start:** With at-start LID in `Latency` mode the result is returned in less than 5 seconds. With at-start LID in `Accuracy` mode the result is returned within 30 seconds. You set the priority for at-start LID with the `SpeechServiceConnection_SingleLanguageIdPriority` property.
* **Continuous:** With continuous LID in `Latency` mode the results are returned every 2 seconds for the duration of the audio. With continuous LID in `Accuracy` mode the results are returned within no set time frame for the duration of the audio. You set the priority for continuous LID with the `SpeechServiceConnection_ContinuousLanguageIdPriority` property.

> [!IMPORTANT]
> With [speech-to-text](#speech-to-text) and [speech translation](#speech-translation) continuous recognition, do not set `Accuracy`with the SpeechServiceConnection_ContinuousLanguageIdPriority property. The setting will be ignored without error, and the default priority of `Latency` will remain in effect. Only [standalone language identification](#standalone-language-identification) supports continuous LID with `Accuracy` prioritization.  
Speech uses at-start LID with `Latency` prioritization by default. You need to set a priority property for any other LID configuration.

::: zone pivot="programming-language-csharp"
Here is an example of using continuous LID while still prioritizing latency.

```csharp
speechConfig.SetProperty(PropertyId.SpeechServiceConnection_ContinuousLanguageIdPriority, "Latency");
```

::: zone-end
::: zone pivot="programming-language-cpp"
Here is an example of using continuous LID while still prioritizing latency.

```cpp
speechConfig->SetProperty(PropertyId::SpeechServiceConnection_ContinuousLanguageIdPriority, "Latency");
```

::: zone-end
::: zone pivot="programming-language-python"
Here is an example of using continuous LID while still prioritizing latency.

```python
speech_config.set_property(property_id=speechsdk.PropertyId.SpeechServiceConnection_ContinuousLanguageIdPriority, value='Latency')
```

::: zone-end

When prioritizing `Latency`, the Speech service returns one of the candidate languages provided even if those languages were not in the audio. For example, if `fr-FR` (French) and `en-US` (English) are provided as candidates, but German is spoken, either `fr-FR` or `en-US` would be returned. When prioritizing `Accuracy`, the Speech service will return the string `Unknown` as the detected language if none of the candidate languages are detected or if the language identification confidence is low.

> [!NOTE]
> You may see cases where an empty string will be returned instead of `Unknown`, due to Speech service inconsistency.
> While this note is present, applications should check for both the `Unknown` and empty string case and treat them identically.
### Recognize once or continuous

Language identification is completed with recognition objects and operations. You will make a request to the Speech service for recognition of audio.

> [!NOTE]
> Don't confuse recognition with identification. Recognition can be used with or without language identification.
Let's map these concepts to the code. You will either call the recognize once method, or the start and stop continuous recognition methods. You choose from:

- Recognize once with at-start LID
- Continuous recognition with at start LID
- Continuous recognition with continuous LID

The `SpeechServiceConnection_ContinuousLanguageIdPriority` property is always required for continuous LID. Without it the speech service defaults to at-start lid.

::: zone pivot="programming-language-csharp"

```csharp
// Recognize once with At-start LID
var result = await recognizer.RecognizeOnceAsync();

// Start and stop continuous recognition with At-start LID
await recognizer.StartContinuousRecognitionAsync();
await recognizer.StopContinuousRecognitionAsync();

// Start and stop continuous recognition with Continuous LID
speechConfig.SetProperty(PropertyId.SpeechServiceConnection_ContinuousLanguageIdPriority, "Latency");
await recognizer.StartContinuousRecognitionAsync();
await recognizer.StopContinuousRecognitionAsync();
```

::: zone-end
::: zone pivot="programming-language-cpp"

```cpp
// Recognize once with At-start LID
auto result = recognizer->RecognizeOnceAsync().get();

// Start and stop continuous recognition with At-start LID
recognizer->StartContinuousRecognitionAsync().get();
recognizer->StopContinuousRecognitionAsync().get();

// Start and stop continuous recognition with Continuous LID
speechConfig->SetProperty(PropertyId::SpeechServiceConnection_ContinuousLanguageIdPriority, "Latency");
recognizer->StartContinuousRecognitionAsync().get();
recognizer->StopContinuousRecognitionAsync().get();
```

::: zone-end
::: zone pivot="programming-language-python"

```python
# Recognize once with At-start LID
result = recognizer.recognize_once()

# Start and stop continuous recognition with At-start LID
source_language_recognizer.start_continuous_recognition()
source_language_recognizer.stop_continuous_recognition()

# Start and stop continuous recognition with Continuous LID
speech_config.set_property(property_id=speechsdk.PropertyId.SpeechServiceConnection_ContinuousLanguageIdPriority, value='Latency')
source_language_recognizer.start_continuous_recognition()
source_language_recognizer.stop_continuous_recognition()
```

::: zone-end

## Standalone language identification

You use standalone language identification when you only need to identify the language in an audio source.

> [!NOTE]
> Standalone source language identification is only supported with the Speech SDKs for C#, C++, and Python.
::: zone pivot="programming-language-csharp"

### [Recognize once](#tab/once)

:::code language="csharp" source="~/samples-cognitive-services-speech-sdk/samples/csharp/sharedcontent/console/standalone_language_detection_samples.cs" id="languageDetectionInAccuracyWithFile":::

### [Continuous recognition](#tab/continuous)

:::code language="csharp" source="~/samples-cognitive-services-speech-sdk/samples/csharp/sharedcontent/console/standalone_language_detection_samples.cs" id="languageDetectionContinuousWithFile":::

---

See more examples of standalone language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/csharp/sharedcontent/console/standalone_language_detection_samples.cs).

::: zone-end

::: zone pivot="programming-language-cpp"

### [Recognize once](#tab/once)

:::code language="cpp" source="~/samples-cognitive-services-speech-sdk/samples/cpp/windows/console/samples/standalone_language_detection_samples.cpp" id="StandaloneLanguageDetectionWithMicrophone":::

### [Continuous recognition](#tab/continuous)

:::code language="cpp" source="~/samples-cognitive-services-speech-sdk/samples/cpp/windows/console/samples/standalone_language_detection_samples.cpp" id="StandaloneLanguageDetectionInContinuousModeWithFileInput":::

---

See more examples of standalone language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/cpp/windows/console/samples/standalone_language_detection_samples.cpp).

::: zone-end

::: zone pivot="programming-language-python"

### [Recognize once](#tab/once)

:::code language="python" source="~/samples-cognitive-services-speech-sdk/samples/python/console/speech_language_detection_sample.py" id="SpeechLanguageDetectionWithFile":::

### [Continuous recognition](#tab/continuous)

:::code language="python" source="~/samples-cognitive-services-speech-sdk/samples/python/console/speech_language_detection_sample.py" id="SpeechContinuousLanguageDetectionWithFile":::

---

See more examples of standalone language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/python/console/speech_language_detection_sample.py).

::: zone-end

## Speech-to-text

You use Speech-to-text recognition when you need to identify the language in an audio source and then transcribe it to text. For more information, see [Speech-to-text overview](speech-to-text.md).

> [!NOTE]
> Speech-to-text recognition with at-start language identification is supported with Speech SDKs in C#, C++, Python, Java, JavaScript, and Objective-C. Speech-to-text recognition with continuous language identification is only supported with Speech SDKs in C#, C++, and Python.
> Currently for speech-to-text recognition with continuous language identification, you must create a SpeechConfig from the `wss://{region}.stt.speech.microsoft.com/speech/universal/v2` endpoint string, as shown in code examples. In a future SDK release you won't need to set it.
::: zone pivot="programming-language-csharp"

### [Recognize once](#tab/once)

```csharp
using Microsoft.CognitiveServices.Speech;
using Microsoft.CognitiveServices.Speech.Audio;

var speechConfig = SpeechConfig.FromSubscription("YourSubscriptionKey","YourServiceRegion");

speechConfig.SetProperty(PropertyId.SpeechServiceConnection_SingleLanguageIdPriority, "Latency");

var autoDetectSourceLanguageConfig =
    AutoDetectSourceLanguageConfig.FromLanguages(
        new string[] { "en-US", "de-DE", "zh-CN" });

using var audioConfig = AudioConfig.FromDefaultMicrophoneInput();
using (var recognizer = new SpeechRecognizer(
    speechConfig,
    autoDetectSourceLanguageConfig,
    audioConfig))
{
    var speechRecognitionResult = await recognizer.RecognizeOnceAsync();
    var autoDetectSourceLanguageResult =
        AutoDetectSourceLanguageResult.FromResult(speechRecognitionResult);
    var detectedLanguage = autoDetectSourceLanguageResult.Language;
}
```

### [Continuous recognition](#tab/continuous)

```csharp
using Microsoft.CognitiveServices.Speech;
using Microsoft.CognitiveServices.Speech.Audio;

var region = "YourServiceRegion";
// Currently the v2 endpoint is required. In a future SDK release you won't need to set it.
var endpointString = $"wss://{region}.stt.speech.microsoft.com/speech/universal/v2";
var endpointUrl = new Uri(endpointString);

var config = SpeechConfig.FromEndpoint(endpointUrl, "YourSubscriptionKey");
// can switch "Latency" to "Accuracy" depending on priority
config.SetProperty(PropertyId.SpeechServiceConnection_ContinuousLanguageIdPriority, "Latency");

var autoDetectSourceLanguageConfig = AutoDetectSourceLanguageConfig.FromLanguages(new string[] { "en-US", "de-DE", "zh-CN" });

var stopRecognition = new TaskCompletionSource<int>();
using (var audioInput = AudioConfig.FromWavFileInput(@"en-us_zh-cn.wav"))
{
    using (var recognizer = new SpeechRecognizer(config, autoDetectSourceLanguageConfig, audioInput))
    {
        // Subscribes to events.
        recognizer.Recognizing += (s, e) =>
        {
            if (e.Result.Reason == ResultReason.RecognizingSpeech)
            {
                Console.WriteLine($"RECOGNIZING: Text={e.Result.Text}");
                var autoDetectSourceLanguageResult = AutoDetectSourceLanguageResult.FromResult(e.Result);
                Console.WriteLine($"DETECTED: Language={autoDetectSourceLanguageResult.Language}");
            }
        };

        recognizer.Recognized += (s, e) =>
        {
            if (e.Result.Reason == ResultReason.RecognizedSpeech)
            {
                Console.WriteLine($"RECOGNIZED: Text={e.Result.Text}");
                var autoDetectSourceLanguageResult = AutoDetectSourceLanguageResult.FromResult(e.Result);
                Console.WriteLine($"DETECTED: Language={autoDetectSourceLanguageResult.Language}");
            }
            else if (e.Result.Reason == ResultReason.NoMatch)
            {
                Console.WriteLine($"NOMATCH: Speech could not be recognized.");
            }
        };

        recognizer.Canceled += (s, e) =>
        {
            Console.WriteLine($"CANCELED: Reason={e.Reason}");

            if (e.Reason == CancellationReason.Error)
            {
                Console.WriteLine($"CANCELED: ErrorCode={e.ErrorCode}");
                Console.WriteLine($"CANCELED: ErrorDetails={e.ErrorDetails}");
                Console.WriteLine($"CANCELED: Did you update the subscription info?");
            }

            stopRecognition.TrySetResult(0);
        };

        recognizer.SessionStarted += (s, e) =>
        {
            Console.WriteLine("\n    Session started event.");
        };

        recognizer.SessionStopped += (s, e) =>
        {
            Console.WriteLine("\n    Session stopped event.");
            Console.WriteLine("\nStop recognition.");
            stopRecognition.TrySetResult(0);
        };

        // Starts continuous recognition. Uses StopContinuousRecognitionAsync() to stop recognition.
        await recognizer.StartContinuousRecognitionAsync().ConfigureAwait(false);

        // Waits for completion.
        // Use Task.WaitAny to keep the task rooted.
        Task.WaitAny(new[] { stopRecognition.Task });

        // Stops recognition.
        await recognizer.StopContinuousRecognitionAsync().ConfigureAwait(false);
    }
}
```

See more examples of speech-to-text recognition with language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/csharp/sharedcontent/console/translation_samples.cs).

::: zone-end

::: zone pivot="programming-language-cpp"

### [Recognize once](#tab/once)

```cpp
using namespace std;
using namespace Microsoft::CognitiveServices::Speech;
using namespace Microsoft::CognitiveServices::Speech::Audio;

auto speechConfig = SpeechConfig::FromSubscription("YourSubscriptionKey","YourServiceRegion");
speechConfig->SetProperty(PropertyId::SpeechServiceConnection_SingleLanguageIdPriority, "Latency");

auto autoDetectSourceLanguageConfig =
    AutoDetectSourceLanguageConfig::FromLanguages({ "en-US", "de-DE", "zh-CN" });

auto recognizer = SpeechRecognizer::FromConfig(
    speechConfig,
    autoDetectSourceLanguageConfig
    );

speechRecognitionResult = recognizer->RecognizeOnceAsync().get();
auto autoDetectSourceLanguageResult =
    AutoDetectSourceLanguageResult::FromResult(speechRecognitionResult);
auto detectedLanguage = autoDetectSourceLanguageResult->Language;
```

### [Continuous recognition](#tab/continuous)

:::code language="cpp" source="~/samples-cognitive-services-speech-sdk/samples/cpp/windows/console/samples/speech_recognition_samples.cpp" id="SpeechContinuousRecognitionAndLanguageIdWithMultiLingualFile":::

See more examples of speech-to-text recognition with language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/cpp/windows/console/samples/speech_recognition_samples.cpp).

::: zone-end

::: zone pivot="programming-language-java"

```java
AutoDetectSourceLanguageConfig autoDetectSourceLanguageConfig =
    AutoDetectSourceLanguageConfig.fromLanguages(Arrays.asList("en-US", "de-DE"));

SpeechRecognizer recognizer = new SpeechRecognizer(
    speechConfig,
    autoDetectSourceLanguageConfig,
    audioConfig);

Future<SpeechRecognitionResult> future = recognizer.recognizeOnceAsync();
SpeechRecognitionResult result = future.get(30, TimeUnit.SECONDS);
AutoDetectSourceLanguageResult autoDetectSourceLanguageResult =
    AutoDetectSourceLanguageResult.fromResult(result);
String detectedLanguage = autoDetectSourceLanguageResult.getLanguage();

recognizer.close();
speechConfig.close();
autoDetectSourceLanguageConfig.close();
audioConfig.close();
result.close();
```

See more examples of speech-to-text recognition with language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/java/jre/console/src/com/microsoft/cognitiveservices/speech/samples/console/SpeechRecognitionSamples.java).

::: zone-end

::: zone pivot="programming-language-python"

### [Recognize once](#tab/once)

```Python
auto_detect_source_language_config = \
        speechsdk.languageconfig.AutoDetectSourceLanguageConfig(languages=["en-US", "de-DE"])
speech_recognizer = speechsdk.SpeechRecognizer(
        speech_config=speech_config, 
        auto_detect_source_language_config=auto_detect_source_language_config, 
        audio_config=audio_config)
result = speech_recognizer.recognize_once()
auto_detect_source_language_result = speechsdk.AutoDetectSourceLanguageResult(result)
detected_language = auto_detect_source_language_result.language
```

### [Continuous recognition](#tab/continuous)

```python
import azure.cognitiveservices.speech as speechsdk
import time
import json

speech_key, service_region = "YourSubscriptionKey","YourServiceRegion"
weatherfilename="en-us_zh-cn.wav"

# Currently the v2 endpoint is required. In a future SDK release you won't need to set it. 
endpoint_string = "wss://{}.stt.speech.microsoft.com/speech/universal/v2".format(service_region)
speech_config = speechsdk.SpeechConfig(subscription=speech_key, endpoint=endpoint_string)
audio_config = speechsdk.audio.AudioConfig(filename=weatherfilename)

auto_detect_source_language_config = speechsdk.languageconfig.AutoDetectSourceLanguageConfig(
    languages=["en-US", "de-DE", "zh-CN"])

speech_recognizer = speechsdk.SpeechRecognizer(
    speech_config=speech_config, 
    auto_detect_source_language_config=auto_detect_source_language_config,
    audio_config=audio_config)

done = False

def stop_cb(evt):
    """callback that signals to stop continuous recognition upon receiving an event `evt`"""
    print('CLOSING on {}'.format(evt))
    nonlocal done
    done = True

# Connect callbacks to the events fired by the speech recognizer
speech_recognizer.recognizing.connect(lambda evt: print('RECOGNIZING: {}'.format(evt)))
speech_recognizer.recognized.connect(lambda evt: print('RECOGNIZED: {}'.format(evt)))
speech_recognizer.session_started.connect(lambda evt: print('SESSION STARTED: {}'.format(evt)))
speech_recognizer.session_stopped.connect(lambda evt: print('SESSION STOPPED {}'.format(evt)))
speech_recognizer.canceled.connect(lambda evt: print('CANCELED {}'.format(evt)))
# stop continuous recognition on either session stopped or canceled events
speech_recognizer.session_stopped.connect(stop_cb)
speech_recognizer.canceled.connect(stop_cb)

# Start continuous speech recognition
speech_recognizer.start_continuous_recognition()
while not done:
    time.sleep(.5)

speech_recognizer.stop_continuous_recognition()
```

See more examples of speech-to-text recognition with language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/python/console/speech_sample.py).

::: zone-end

::: zone pivot="programming-language-objectivec"

```Objective-C
NSArray *languages = @[@"en-US", @"de-DE", @"zh-CN"];
SPXAutoDetectSourceLanguageConfiguration* autoDetectSourceLanguageConfig = \
        [[SPXAutoDetectSourceLanguageConfiguration alloc]init:languages];
SPXSpeechRecognizer* speechRecognizer = \
        [[SPXSpeechRecognizer alloc] initWithSpeechConfiguration:speechConfig
                           autoDetectSourceLanguageConfiguration:autoDetectSourceLanguageConfig
                                              audioConfiguration:audioConfig];
SPXSpeechRecognitionResult *result = [speechRecognizer recognizeOnce];
SPXAutoDetectSourceLanguageResult *languageDetectionResult = [[SPXAutoDetectSourceLanguageResult alloc] init:result];
NSString *detectedLanguage = [languageDetectionResult language];
```

::: zone-end

::: zone pivot="programming-language-javascript"

```Javascript
var autoDetectSourceLanguageConfig = SpeechSDK.AutoDetectSourceLanguageConfig.fromLanguages(["en-US", "de-DE"]);
var speechRecognizer = SpeechSDK.SpeechRecognizer.FromConfig(speechConfig, autoDetectSourceLanguageConfig, audioConfig);
speechRecognizer.recognizeOnceAsync((result: SpeechSDK.SpeechRecognitionResult) => {
        var languageDetectionResult = SpeechSDK.AutoDetectSourceLanguageResult.fromResult(result);
        var detectedLanguage = languageDetectionResult.language;
},
{});
```

::: zone-end

### Using Speech-to-text custom models

::: zone pivot="programming-language-csharp"
This sample shows how to use language detection with a custom endpoint. If the detected language is `en-US`, then the default model is used. If the detected language is `fr-FR`, then the custom model endpoint is used. For more information, see [Train and deploy a Custom Speech model](how-to-custom-speech-train-model.md).

```csharp
var sourceLanguageConfigs = new SourceLanguageConfig[]
{
    SourceLanguageConfig.FromLanguage("en-US"),
    SourceLanguageConfig.FromLanguage("fr-FR", "The Endpoint Id for custom model of fr-FR")
};
var autoDetectSourceLanguageConfig =
    AutoDetectSourceLanguageConfig.FromSourceLanguageConfigs(
        sourceLanguageConfigs);
```

::: zone-end

::: zone pivot="programming-language-cpp"
This sample shows how to use language detection with a custom endpoint. If the detected language is `en-US`, then the default model is used. If the detected language is `fr-FR`, then the custom model endpoint is used. For more information, see [Train and deploy a Custom Speech model](how-to-custom-speech-train-model.md).

```cpp
std::vector<std::shared_ptr<SourceLanguageConfig>> sourceLanguageConfigs;
sourceLanguageConfigs.push_back(
    SourceLanguageConfig::FromLanguage("en-US"));
sourceLanguageConfigs.push_back(
    SourceLanguageConfig::FromLanguage("fr-FR", "The Endpoint Id for custom model of fr-FR"));

auto autoDetectSourceLanguageConfig =
    AutoDetectSourceLanguageConfig::FromSourceLanguageConfigs(
        sourceLanguageConfigs);
```

::: zone-end

::: zone pivot="programming-language-java"
This sample shows how to use language detection with a custom endpoint. If the detected language is `en-US`, then the default model is used. If the detected language is `fr-FR`, then the custom model endpoint is used. For more information, see [Train and deploy a Custom Speech model](how-to-custom-speech-train-model.md).

```java
List sourceLanguageConfigs = new ArrayList<SourceLanguageConfig>();
sourceLanguageConfigs.add(
    SourceLanguageConfig.fromLanguage("en-US"));
sourceLanguageConfigs.add(
    SourceLanguageConfig.fromLanguage("fr-FR", "The Endpoint Id for custom model of fr-FR"));

AutoDetectSourceLanguageConfig autoDetectSourceLanguageConfig =
    AutoDetectSourceLanguageConfig.fromSourceLanguageConfigs(
        sourceLanguageConfigs);
```

::: zone-end

::: zone pivot="programming-language-python"
This sample shows how to use language detection with a custom endpoint. If the detected language is `en-US`, then the default model is used. If the detected language is `fr-FR`, then the custom model endpoint is used. For more information, see [Train and deploy a Custom Speech model](how-to-custom-speech-train-model.md).

```Python
 en_language_config = speechsdk.languageconfig.SourceLanguageConfig("en-US")
 fr_language_config = speechsdk.languageconfig.SourceLanguageConfig("fr-FR", "The Endpoint Id for custom model of fr-FR")
 auto_detect_source_language_config = speechsdk.languageconfig.AutoDetectSourceLanguageConfig(
        sourceLanguageConfigs=[en_language_config, fr_language_config])
```

::: zone-end

::: zone pivot="programming-language-objectivec"
This sample shows how to use language detection with a custom endpoint. If the detected language is `en-US`, then the default model is used. If the detected language is `fr-FR`, then the custom model endpoint is used. For more information, see [Train and deploy a Custom Speech model](how-to-custom-speech-train-model.md).

```Objective-C
SPXSourceLanguageConfiguration* enLanguageConfig = [[SPXSourceLanguageConfiguration alloc]init:@"en-US"];
SPXSourceLanguageConfiguration* frLanguageConfig = \
        [[SPXSourceLanguageConfiguration alloc]initWithLanguage:@"fr-FR"
                                                     endpointId:@"The Endpoint Id for custom model of fr-FR"];
NSArray *languageConfigs = @[enLanguageConfig, frLanguageConfig];
SPXAutoDetectSourceLanguageConfiguration* autoDetectSourceLanguageConfig = \
        [[SPXAutoDetectSourceLanguageConfiguration alloc]initWithSourceLanguageConfigurations:languageConfigs];
```

::: zone-end

::: zone pivot="programming-language-javascript"
Language detection with a custom endpoint is not supported by the Speech SDK for JavaScript. For example, if you include "fr-FR" as shown here, the custom endpoint will be ignored.

```Javascript
var enLanguageConfig = SpeechSDK.SourceLanguageConfig.fromLanguage("en-US");
var frLanguageConfig = SpeechSDK.SourceLanguageConfig.fromLanguage("fr-FR", "The Endpoint Id for custom model of fr-FR");
var autoDetectSourceLanguageConfig = SpeechSDK.AutoDetectSourceLanguageConfig.fromSourceLanguageConfigs([enLanguageConfig, frLanguageConfig]);
```

::: zone-end

## Speech translation

You use Speech translation when you need to identify the language in an audio source and then translate it to another language. For more information, see [Speech translation overview](speech-translation.md).

> [!NOTE]
> Speech translation with language identification is only supported with Speech SDKs in C#, C++, and Python. 
> Currently for speech translation with language identification, you must create a SpeechConfig from the `wss://{region}.stt.speech.microsoft.com/speech/universal/v2` endpoint string, as shown in code examples. In a future SDK release you won't need to set it.
::: zone pivot="programming-language-csharp"

### [Recognize once](#tab/once)

```csharp
using Microsoft.CognitiveServices.Speech;
using Microsoft.CognitiveServices.Speech.Audio;
using Microsoft.CognitiveServices.Speech.Translation;

public static async Task RecognizeOnceSpeechTranslationAsync()
{
    var region = "YourServiceRegion";
    // Currently the v2 endpoint is required. In a future SDK release you won't need to set it.
    var endpointString = $"wss://{region}.stt.speech.microsoft.com/speech/universal/v2";
    var endpointUrl = new Uri(endpointString);
    
    var config = SpeechTranslationConfig.FromEndpoint(endpointUrl, "YourSubscriptionKey");

    speechTranslationConfig.SetProperty(PropertyId.SpeechServiceConnection_SingleLanguageIdPriority, "Latency");

    // Source language is required, but currently ignored. 
    string fromLanguage = "en-US";
    speechTranslationConfig.SpeechRecognitionLanguage = fromLanguage;

    speechTranslationConfig.AddTargetLanguage("de");
    speechTranslationConfig.AddTargetLanguage("fr");

    var autoDetectSourceLanguageConfig = AutoDetectSourceLanguageConfig.FromLanguages(new string[] { "en-US", "de-DE", "zh-CN" });

    using var audioConfig = AudioConfig.FromDefaultMicrophoneInput();

    using (var recognizer = new TranslationRecognizer(
        speechTranslationConfig, 
        autoDetectSourceLanguageConfig,
        audioConfig))
    {
        
        Console.WriteLine("Say something or read from file...");
        var result = await recognizer.RecognizeOnceAsync().ConfigureAwait(false);

        if (result.Reason == ResultReason.TranslatedSpeech)
        {
            var lidResult = result.Properties.GetProperty(PropertyId.SpeechServiceConnection_AutoDetectSourceLanguageResult);

            Console.WriteLine($"RECOGNIZED in '{lidResult}': Text={result.Text}");
            foreach (var element in result.Translations)
            {
                Console.WriteLine($"    TRANSLATED into '{element.Key}': {element.Value}");
            }
        }
    }
}
```

### [Continuous recognition](#tab/continuous)

```csharp
using Microsoft.CognitiveServices.Speech;
using Microsoft.CognitiveServices.Speech.Audio;
using Microsoft.CognitiveServices.Speech.Translation;

public static async Task MultiLingualTranslation()
{
    var region = "YourServiceRegion";
    // Currently the v2 endpoint is required. In a future SDK release you won't need to set it.
    var endpointString = $"wss://{region}.stt.speech.microsoft.com/speech/universal/v2";
    var endpointUrl = new Uri(endpointString);
    
    var config = SpeechTranslationConfig.FromEndpoint(endpointUrl, "YourSubscriptionKey");

    // Source language is required, but currently ignored. 
    string fromLanguage = "en-US";
    config.SpeechRecognitionLanguage = fromLanguage;

    config.AddTargetLanguage("de");
    config.AddTargetLanguage("fr");

    config.SetProperty(PropertyId.SpeechServiceConnection_ContinuousLanguageIdPriority, "Latency");
    var autoDetectSourceLanguageConfig = AutoDetectSourceLanguageConfig.FromLanguages(new string[] { "en-US", "de-DE", "zh-CN" });

    var stopTranslation = new TaskCompletionSource<int>();
    using (var audioInput = AudioConfig.FromWavFileInput(@"en-us_zh-cn.wav"))
    {
        using (var recognizer = new TranslationRecognizer(config, autoDetectSourceLanguageConfig, audioInput))
        {
            recognizer.Recognizing += (s, e) =>
            {
                var lidResult = e.Result.Properties.GetProperty(PropertyId.SpeechServiceConnection_AutoDetectSourceLanguageResult);

                Console.WriteLine($"RECOGNIZING in '{lidResult}': Text={e.Result.Text}");
                foreach (var element in e.Result.Translations)
                {
                    Console.WriteLine($"    TRANSLATING into '{element.Key}': {element.Value}");
                }
            };

            recognizer.Recognized += (s, e) => {
                if (e.Result.Reason == ResultReason.TranslatedSpeech)
                {
                    var lidResult = e.Result.Properties.GetProperty(PropertyId.SpeechServiceConnection_AutoDetectSourceLanguageResult);

                    Console.WriteLine($"RECOGNIZED in '{lidResult}': Text={e.Result.Text}");
                    foreach (var element in e.Result.Translations)
                    {
                        Console.WriteLine($"    TRANSLATED into '{element.Key}': {element.Value}");
                    }
                }
                else if (e.Result.Reason == ResultReason.RecognizedSpeech)
                {
                    Console.WriteLine($"RECOGNIZED: Text={e.Result.Text}");
                    Console.WriteLine($"    Speech not translated.");
                }
                else if (e.Result.Reason == ResultReason.NoMatch)
                {
                    Console.WriteLine($"NOMATCH: Speech could not be recognized.");
                }
            };

            recognizer.Canceled += (s, e) =>
            {
                Console.WriteLine($"CANCELED: Reason={e.Reason}");

                if (e.Reason == CancellationReason.Error)
                {
                    Console.WriteLine($"CANCELED: ErrorCode={e.ErrorCode}");
                    Console.WriteLine($"CANCELED: ErrorDetails={e.ErrorDetails}");
                    Console.WriteLine($"CANCELED: Did you update the subscription info?");
                }

                stopTranslation.TrySetResult(0);
            };

            recognizer.SpeechStartDetected += (s, e) => {
                Console.WriteLine("\nSpeech start detected event.");
            };

            recognizer.SpeechEndDetected += (s, e) => {
                Console.WriteLine("\nSpeech end detected event.");
            };

            recognizer.SessionStarted += (s, e) => {
                Console.WriteLine("\nSession started event.");
            };

            recognizer.SessionStopped += (s, e) => {
                Console.WriteLine("\nSession stopped event.");
                Console.WriteLine($"\nStop translation.");
                stopTranslation.TrySetResult(0);
            };

            // Starts continuous recognition. Uses StopContinuousRecognitionAsync() to stop recognition.
            Console.WriteLine("Start translation...");
            await recognizer.StartContinuousRecognitionAsync().ConfigureAwait(false);

            Task.WaitAny(new[] { stopTranslation.Task });
            await recognizer.StopContinuousRecognitionAsync().ConfigureAwait(false);
        }
    }
}
```

See more examples of speech translation with language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/csharp/sharedcontent/console/translation_samples.cs).

::: zone-end

::: zone pivot="programming-language-cpp"

### [Recognize once](#tab/once)

```cpp
auto region = "YourServiceRegion";
// Currently the v2 endpoint is required. In a future SDK release you won't need to set it.
auto endpointString = std::format("wss://{}.stt.speech.microsoft.com/speech/universal/v2", region);
auto config = SpeechTranslationConfig::FromEndpoint(endpointString, "YourSubscriptionKey");

// Language Id feature requirement
// Please refer to language id document for different modes
config->SetProperty(PropertyId::SpeechServiceConnection_SingleLanguageIdPriority, "Latency");
auto autoDetectSourceLanguageConfig = AutoDetectSourceLanguageConfig::FromLanguages({ "en-US", "de-DE" });

// Sets source and target languages
// The source language will be detected by the language detection feature. 
// However, the SpeechRecognitionLanguage still need to set with a locale string, but it will not be used as the source language.
// This will be fixed in a future version of Speech SDK.
auto fromLanguage = "en-US";
config->SetSpeechRecognitionLanguage(fromLanguage);
config->AddTargetLanguage("de");
config->AddTargetLanguage("fr");

// Creates a translation recognizer using microphone as audio input.
auto recognizer = TranslationRecognizer::FromConfig(config, autoDetectSourceLanguageConfig);
cout << "Say something...\n";

// Starts translation, and returns after a single utterance is recognized. The end of a
// single utterance is determined by listening for silence at the end or until a maximum of 15
// seconds of audio is processed. The task returns the recognized text as well as the translation.
// Note: Since RecognizeOnceAsync() returns only a single utterance, it is suitable only for single
// shot recognition like command or query.
// For long-running multi-utterance recognition, use StartContinuousRecognitionAsync() instead.
auto result = recognizer->RecognizeOnceAsync().get();

// Checks result.
if (result->Reason == ResultReason::TranslatedSpeech)
{
    cout << "RECOGNIZED: Text=" << result->Text << std::endl;

    for (const auto& it : result->Translations)
    {
        cout << "TRANSLATED into '" << it.first.c_str() << "': " << it.second.c_str() << std::endl;
    }
}
else if (result->Reason == ResultReason::RecognizedSpeech)
{
    cout << "RECOGNIZED: Text=" << result->Text << " (text could not be translated)" << std::endl;
}
else if (result->Reason == ResultReason::NoMatch)
{
    cout << "NOMATCH: Speech could not be recognized." << std::endl;
}
else if (result->Reason == ResultReason::Canceled)
{
    auto cancellation = CancellationDetails::FromResult(result);
    cout << "CANCELED: Reason=" << (int)cancellation->Reason << std::endl;

    if (cancellation->Reason == CancellationReason::Error)
    {
        cout << "CANCELED: ErrorCode=" << (int)cancellation->ErrorCode << std::endl;
        cout << "CANCELED: ErrorDetails=" << cancellation->ErrorDetails << std::endl;
        cout << "CANCELED: Did you update the subscription info?" << std::endl;
    }
}
```

### [Continuous recognition](#tab/continuous)

```cpp
using namespace std;
using namespace Microsoft::CognitiveServices::Speech;
using namespace Microsoft::CognitiveServices::Speech::Audio;
using namespace Microsoft::CognitiveServices::Speech::Translation;

void MultiLingualTranslation()
{
    auto region = "YourServiceRegion";
    // Currently the v2 endpoint is required. In a future SDK release you won't need to set it.
    auto endpointString = std::format("wss://{}.stt.speech.microsoft.com/speech/universal/v2", region);
    auto config = SpeechTranslationConfig::FromEndpoint(endpointString, "YourSubscriptionKey");

    speechConfig->SetProperty(PropertyId::SpeechServiceConnection_ContinuousLanguageIdPriority, "Latency");
    auto autoDetectSourceLanguageConfig = AutoDetectSourceLanguageConfig::FromLanguages({ "en-US", "de-DE", "zh-CN" });

    promise<void> recognitionEnd;
    // Source language is required, but currently ignored. 
    auto fromLanguage = "en-US";
    config->SetSpeechRecognitionLanguage(fromLanguage);
    config->AddTargetLanguage("de");
    config->AddTargetLanguage("fr");

    auto audioInput = AudioConfig::FromWavFileInput("whatstheweatherlike.wav");
    auto recognizer = TranslationRecognizer::FromConfig(config, autoDetectSourceLanguageConfig, audioInput);

    recognizer->Recognizing.Connect([](const TranslationRecognitionEventArgs& e)
        {
            std::string lidResult = e.Result->Properties.GetProperty(PropertyId::SpeechServiceConnection_AutoDetectSourceLanguageResult);

            cout << "Recognizing in Language = "<< lidResult << ":" << e.Result->Text << std::endl;
            for (const auto& it : e.Result->Translations)
            {
                cout << "  Translated into '" << it.first.c_str() << "': " << it.second.c_str() << std::endl;
            }
        });

    recognizer->Recognized.Connect([](const TranslationRecognitionEventArgs& e)
        {
            if (e.Result->Reason == ResultReason::TranslatedSpeech)
            {
                std::string lidResult = e.Result->Properties.GetProperty(PropertyId::SpeechServiceConnection_AutoDetectSourceLanguageResult);
                cout << "RECOGNIZED in Language = " << lidResult << ": Text=" << e.Result->Text << std::endl;
            }
            else if (e.Result->Reason == ResultReason::RecognizedSpeech)
            {
                cout << "RECOGNIZED: Text=" << e.Result->Text << " (text could not be translated)" << std::endl;
            }
            else if (e.Result->Reason == ResultReason::NoMatch)
            {
                cout << "NOMATCH: Speech could not be recognized." << std::endl;
            }

            for (const auto& it : e.Result->Translations)
            {
                cout << "  Translated into '" << it.first.c_str() << "': " << it.second.c_str() << std::endl;
            }
        });

    recognizer->Canceled.Connect([&recognitionEnd](const TranslationRecognitionCanceledEventArgs& e)
        {
            cout << "CANCELED: Reason=" << (int)e.Reason << std::endl;
            if (e.Reason == CancellationReason::Error)
            {
                cout << "CANCELED: ErrorCode=" << (int)e.ErrorCode << std::endl;
                cout << "CANCELED: ErrorDetails=" << e.ErrorDetails << std::endl;
                cout << "CANCELED: Did you update the subscription info?" << std::endl;

                recognitionEnd.set_value();
            }
        });

    recognizer->Synthesizing.Connect([](const TranslationSynthesisEventArgs& e)
        {
            auto size = e.Result->Audio.size();
            cout << "Translation synthesis result: size of audio data: " << size
                << (size == 0 ? "(END)" : "");
        });

    recognizer->SessionStopped.Connect([&recognitionEnd](const SessionEventArgs& e)
        {
            cout << "Session stopped.";
            recognitionEnd.set_value();
        });

    // Starts continuos recognition. Use StopContinuousRecognitionAsync() to stop recognition.
    recognizer->StartContinuousRecognitionAsync().get();
    recognitionEnd.get_future().get();
    recognizer->StopContinuousRecognitionAsync().get();
}
```

See more examples of speech translation with language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/cpp/windows/console/samples/translation_samples.cpp).

::: zone-end

::: zone pivot="programming-language-python"

### [Recognize once](#tab/once)

```python
import azure.cognitiveservices.speech as speechsdk
import time
import json

speech_key, service_region = "YourSubscriptionKey","YourServiceRegion"
weatherfilename="en-us_zh-cn.wav"

# set up translation parameters: source language and target languages
# Currently the v2 endpoint is required. In a future SDK release you won't need to set it. 
endpoint_string = "wss://{}.stt.speech.microsoft.com/speech/universal/v2".format(service_region)
translation_config = speechsdk.translation.SpeechTranslationConfig(
    subscription=speech_key,
    endpoint=endpoint_string,
    speech_recognition_language='en-US',
    target_languages=('de', 'fr'))
audio_config = speechsdk.audio.AudioConfig(filename=weatherfilename)

# Set the Priority (optional, default Latency, either Latency or Accuracy is accepted)
translation_config.set_property(property_id=speechsdk.PropertyId.SpeechServiceConnection_SingleLanguageIdPriority, value='Accuracy')

# Specify the AutoDetectSourceLanguageConfig, which defines the number of possible languages
auto_detect_source_language_config = speechsdk.languageconfig.AutoDetectSourceLanguageConfig(languages=["en-US", "de-DE", "zh-CN"])

# Creates a translation recognizer using and audio file as input.
recognizer = speechsdk.translation.TranslationRecognizer(
    translation_config=translation_config, 
    audio_config=audio_config,
    auto_detect_source_language_config=auto_detect_source_language_config)

# Starts translation, and returns after a single utterance is recognized. The end of a
# single utterance is determined by listening for silence at the end or until a maximum of 15
# seconds of audio is processed. The task returns the recognition text as result.
# Note: Since recognize_once() returns only a single utterance, it is suitable only for single
# shot recognition like command or query.
# For long-running multi-utterance recognition, use start_continuous_recognition() instead.
result = recognizer.recognize_once()

# Check the result
if result.reason == speechsdk.ResultReason.TranslatedSpeech:
    print("""Recognized: {}
    German translation: {}
    French translation: {}""".format(
        result.text, result.translations['de'], result.translations['fr']))
elif result.reason == speechsdk.ResultReason.RecognizedSpeech:
    print("Recognized: {}".format(result.text))
    detectedSrcLang = result.properties[speechsdk.PropertyId.SpeechServiceConnection_AutoDetectSourceLanguageResult]
    print("Detected Language: {}".format(detectedSrcLang))
elif result.reason == speechsdk.ResultReason.NoMatch:
    print("No speech could be recognized: {}".format(result.no_match_details))
elif result.reason == speechsdk.ResultReason.Canceled:
    print("Translation canceled: {}".format(result.cancellation_details.reason))
    if result.cancellation_details.reason == speechsdk.CancellationReason.Error:
        print("Error details: {}".format(result.cancellation_details.error_details))
```

### [Continuous recognition](#tab/continuous)

```python
import azure.cognitiveservices.speech as speechsdk
import time
import json

speech_key, service_region = "YourSubscriptionKey","YourServiceRegion"
weatherfilename="en-us_zh-cn.wav"

# Currently the v2 endpoint is required. In a future SDK release you won't need to set it. 
endpoint_string = "wss://{}.stt.speech.microsoft.com/speech/universal/v2".format(service_region)
translation_config = speechsdk.translation.SpeechTranslationConfig(
    subscription=speech_key,
    endpoint=endpoint_string,
    speech_recognition_language='en-US',
    target_languages=('de', 'fr'))
audio_config = speechsdk.audio.AudioConfig(filename=weatherfilename)

# Set the Priority (optional, default Latency, either Latency or Accuracy is accepted)
translation_config.set_property(property_id=speechsdk.PropertyId.SpeechServiceConnection_SingleLanguageIdPriority, value='Accuracy')

# Specify the AutoDetectSourceLanguageConfig, which defines the number of possible languages
auto_detect_source_language_config = speechsdk.languageconfig.AutoDetectSourceLanguageConfig(languages=["en-US", "de-DE", "zh-CN"])

# Creates a translation recognizer using and audio file as input.
recognizer = speechsdk.translation.TranslationRecognizer(
    translation_config=translation_config, 
    audio_config=audio_config,
    auto_detect_source_language_config=auto_detect_source_language_config)

def result_callback(event_type, evt):
    """callback to display a translation result"""
    print("{}: {}\n\tTranslations: {}\n\tResult Json: {}".format(
        event_type, evt, evt.result.translations.items(), evt.result.json))

done = False

def stop_cb(evt):
    """callback that signals to stop continuous recognition upon receiving an event `evt`"""
    print('CLOSING on {}'.format(evt))
    nonlocal done
    done = True

# connect callback functions to the events fired by the recognizer
recognizer.session_started.connect(lambda evt: print('SESSION STARTED: {}'.format(evt)))
recognizer.session_stopped.connect(lambda evt: print('SESSION STOPPED {}'.format(evt)))
# event for intermediate results
recognizer.recognizing.connect(lambda evt: result_callback('RECOGNIZING', evt))
# event for final result
recognizer.recognized.connect(lambda evt: result_callback('RECOGNIZED', evt))
# cancellation event
recognizer.canceled.connect(lambda evt: print('CANCELED: {} ({})'.format(evt, evt.reason)))

# stop continuous recognition on either session stopped or canceled events
recognizer.session_stopped.connect(stop_cb)
recognizer.canceled.connect(stop_cb)

def synthesis_callback(evt):
    """
    callback for the synthesis event
    """
    print('SYNTHESIZING {}\n\treceived {} bytes of audio. Reason: {}'.format(
        evt, len(evt.result.audio), evt.result.reason))
    if evt.result.reason == speechsdk.ResultReason.RecognizedSpeech:
        print("RECOGNIZED: {}".format(evt.result.properties))
        if evt.result.properties.get(speechsdk.PropertyId.SpeechServiceConnection_AutoDetectSourceLanguageResult) == None:
            print("Unable to detect any language")
        else:
            detectedSrcLang = evt.result.properties[speechsdk.PropertyId.SpeechServiceConnection_AutoDetectSourceLanguageResult]
            jsonResult = evt.result.properties[speechsdk.PropertyId.SpeechServiceResponse_JsonResult]
            detailResult = json.loads(jsonResult)
            startOffset = detailResult['Offset']
            duration = detailResult['Duration']
            if duration >= 0:
                endOffset = duration + startOffset
            else:
                endOffset = 0
            print("Detected language = " + detectedSrcLang + ", startOffset = " + str(startOffset) + " nanoseconds, endOffset = " + str(endOffset) + " nanoseconds, Duration = " + str(duration) + " nanoseconds.")
            global language_detected
            language_detected = True

# connect callback to the synthesis event
recognizer.synthesizing.connect(synthesis_callback)

# start translation
recognizer.start_continuous_recognition()

while not done:
    time.sleep(.5)

recognizer.stop_continuous_recognition()
```

---

See more examples of speech translation with language identification on [GitHub](https://github.com/Azure-Samples/cognitive-services-speech-sdk/blob/master/samples/python/console/translation_sample.py).

::: zone-end
