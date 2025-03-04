<div align="center">
    <h1 align="center">Self-host Llama 3.1 70B with vLLM and BentoML</h1>
</div>

This is a BentoML example project, showing you how to serve and deploy Llama 3.1 70B (with AWQ quantization) using [vLLM](https://vllm.ai), a high-throughput and memory-efficient inference engine.

See [here](https://docs.bentoml.com/en/latest/examples/overview.html) for a full list of BentoML example projects.

💡 This example is served as a basis for advanced code customization, such as custom model, inference logic or vLLM options. For simple LLM hosting with OpenAI compatible endpoint without writing any code, see [OpenLLM](https://github.com/bentoml/OpenLLM).


## Prerequisites

If you want to test the Service locally, we recommend you use an Nvidia GPU with at least 48G VRAM.

## Install dependencies

```bash
git clone https://github.com/bentoml/BentoVLLM.git
cd BentoVLLM/llama3.1-70b-instruct-awq-function-calling

# Recommend Python 3.11
pip install -r requirements.txt
```

## Run the BentoML Service

We have defined a BentoML Service in `service.py`. Run `bentoml serve` in your project directory to start the Service.

```python
$ bentoml serve .

2024-01-18T07:51:30+0800 [INFO] [cli] Starting production HTTP BentoServer from "service:VLLM" listening on http://localhost:3000 (Press CTRL+C to quit)
INFO 01-18 07:51:40 model_runner.py:501] Capturing the model for CUDA graphs. This may lead to unexpected consequences if the model is not static. To run the model in eager mode, set 'enforce_eager=True' or use '--enforce-eager' in the CLI.
INFO 01-18 07:51:40 model_runner.py:505] CUDA graphs can take additional 1~3 GiB memory per GPU. If you are running out of memory, consider decreasing `gpu_memory_utilization` or enforcing eager mode.
INFO 01-18 07:51:46 model_runner.py:547] Graph capturing finished in 6 secs.
```

The server is now active at [http://localhost:3000](http://localhost:3000/). You can interact with it using the Swagger UI or in other different ways.

<details>

<summary>CURL</summary>

```bash
curl -X 'POST' \
  'http://localhost:3000/generate' \
  -H 'accept: text/event-stream' \
  -H 'Content-Type: application/json' \
  -d '{
  "prompt": "Explain superconductors like I'\''m five years old",
  "tokens": null
}'
```

</details>

<details>

<summary>Python client</summary>

```python
import bentoml

with bentoml.SyncHTTPClient("http://localhost:3000") as client:
    response_generator = client.generate(
        prompt="Explain superconductors like I'm five years old",
        tokens=None
    )
    for response in response_generator:
        print(response)
```

</details>

<details>

<summary>OpenAI-compatible endpoints</summary>

This Service uses the `@openai_endpoints` decorator to set up OpenAI-compatible endpoints (`chat/completions` and `completions`). This means your client can interact with the backend Service (in this case, the VLLM class) as if they were communicating directly with OpenAI's API. This [utility](bentovllm_openai/) does not affect your BentoML Service code, and you can use it for other LLMs as well.

```python
from openai import OpenAI

client = OpenAI(base_url='http://localhost:3000/v1', api_key='na')

# Use the following func to get the available models
client.models.list()

chat_completion = client.chat.completions.create(
    model="hugging-quants/Meta-Llama-3.1-70B-Instruct-AWQ-INT4",
    messages=[
        {
            "role": "user",
            "content": "Explain superconductors like I'm five years old"
        }
    ],
    stream=True,
	stop=["<|eot_id|>", "<|end_of_text|>"],
)
for chunk in chat_completion:
    # Extract and print the content of the model's reply
    print(chunk.choices[0].delta.content or "", end="")
```

These OpenAI-compatible endpoints also support [vLLM extra parameters](https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html#extra-parameters). For example, you can force the chat completion output a JSON object by using the `guided_json` parameters:

```python
from openai import OpenAI

client = OpenAI(base_url='http://localhost:3000/v1', api_key='na')

# Use the following func to get the available models
client.models.list()

json_schema = {
    "type": "object",
    "properties": {
        "city": {"type": "string"}
    }
}

chat_completion = client.chat.completions.create(
    model="hugging-quants/Meta-Llama-3.1-70B-Instruct-AWQ-INT4",
    messages=[
        {
            "role": "user",
            "content": "What is the capital of France?"
        }
    ],
    extra_body=dict(guided_json=json_schema),
)
print(chat_completion.choices[0].message.content)  # will return something like: {"city": "Paris"}
```

The OpenAI-compatible chat endpoint also support auto function calling. To test, run the `test_tool.py` as:

```bash
$ MESSAGE="What's the weather tomorrow in London?" python test_tool.py

ChatCompletion(id='chat-4fca9bc289f0412796a2b3356d5b6fc2', choices=[Choice(finish_reason='tool_calls', index=0, logprobs=None, message=ChatCompletionMessage(content=None, refusal=None, role='assistant', audio=None, function_call=None, tool_calls=[ChatCompletionMessageToolCall(id='chatcmpl-tool-289dc778b6e74ad3a690ba1569509eac', function=Function(arguments='{"location": "London", "format": "celsius", "num_days": "2"}', name='get_n_day_weather_forecast'), type='function')]), stop_reason=128008)], created=1730153141, model='hugging-quants/Meta-Llama-3.1-70B-Instruct-AWQ-INT4', object='chat.completion', service_tier=None, system_fingerprint=None, usage=CompletionUsage(completion_tokens=35, prompt_tokens=317, total_tokens=352, completion_tokens_details=None, prompt_tokens_details=None), prompt_logprobs=None)

$ MESSAGE="Hi!" python test_tool.py

ChatCompletion(id='chat-fa15164af4a74a929835048bde04118d', choices=[Choice(finish_reason='stop', index=0, logprobs=None, message=ChatCompletionMessage(content='I need more information to provide a function call. Can you please provide the location, format, and number of days for which you would like to get the weather forecast?', refusal=None, role='assistant', audio=None, function_call=None, tool_calls=[]), stop_reason=None)], created=1730153182, model='hugging-quants/Meta-Llama-3.1-70B-Instruct-AWQ-INT4', object='chat.completion', service_tier=None, system_fingerprint=None, usage=CompletionUsage(completion_tokens=35, prompt_tokens=310, total_tokens=345, completion_tokens_details=None, prompt_tokens_details=None), prompt_logprobs=None)
```

All supported extra parameters are listed in [vLLM documentation](https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html#extra-parameters).

**Note**: If your Service is deployed with [protected endpoints on BentoCloud](https://docs.bentoml.com/en/latest/bentocloud/how-tos/manage-access-token.html#access-protected-deployments), you need to set the environment variable `OPENAI_API_KEY` to your BentoCloud API key first.

```bash
export OPENAI_API_KEY={YOUR_BENTOCLOUD_API_TOKEN}
```

You can then use the following line to replace the client in the above code snippet. Refer to [Obtain the endpoint URL](https://docs.bentoml.com/en/latest/bentocloud/how-tos/call-deployment-endpoints.html#obtain-the-endpoint-url) to retrieve the endpoint URL.

```python
client = OpenAI(base_url='your_bentocloud_deployment_endpoint_url/v1')
```

</details>

For detailed explanations of the Service code, see [vLLM inference](https://docs.bentoml.org/en/latest/use-cases/large-language-models/vllm.html).

## Deploy to BentoCloud

After the Service is ready, you can deploy the application to BentoCloud for better management and scalability. [Sign up](https://www.bentoml.com/) if you haven't got a BentoCloud account.

Make sure you have [logged in to BentoCloud](https://docs.bentoml.com/en/latest/bentocloud/how-tos/manage-access-token.html), then run the following command to deploy it.

```bash
bentoml deploy .
```

Once the application is up and running on BentoCloud, you can access it via the exposed URL.

**Note**: For custom deployment in your own infrastructure, use [BentoML to generate an OCI-compliant image](https://docs.bentoml.com/en/latest/guides/containerization.html).
