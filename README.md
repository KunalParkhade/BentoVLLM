<div align="center">
    <h1 align="center">Self-host LLMs with vLLM and BentoML</h1>
</div>

This repository contains a group of BentoML example projects, showing you how to serve and deploy open-source Large Language Models using [vLLM](https://vllm.ai), a high-throughput and memory-efficient inference engine. Every model directory contains the code to add OpenAI compatible endpoints to the BentoML Service.

💡 You can use these examples as bases for advanced code customization, such as custom model, inference logic or vLLM options. For simple LLM hosting with OpenAI compatible endpoints without writing any code, see [OpenLLM](https://github.com/bentoml/OpenLLM).

See [here](https://docs.bentoml.com/en/latest/examples/overview.html) for a full list of BentoML example projects.

The following is an example of serving one of the LLMs in this repository: Mistral 7B Instruct.

## Prerequisites

- If you want to test the Service locally, we recommend you use an Nvidia GPU with at least 16G VRAM.
- Gain access to the model in [Hugging Face](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.2).

## Install dependencies

```bash
git clone https://github.com/bentoml/BentoVLLM.git
cd BentoVLLM/mistral-7b-instruct

# Recommend Python 3.11
pip install -r requirements.txt && pip install -f -U "pydantic>=2.0"

export HF_TOEKN=<your-api-key>
```

## Run the BentoML Service

We have defined a BentoML Service in `service.py`. Run `bentoml serve` in your project directory to start the Service.

```bash
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

This Service uses the `@openai_endpoints` decorator to set up OpenAI-compatible endpoints (`chat/completions` and `completions`). This means your client can interact with the backend Service (in this case, the VLLM class) as if they were communicating directly with OpenAI's API. This [utility](mistral-7b-instruct/bentovllm_openai/) does not affect your BentoML Service code, and you can use it for other LLMs as well.

```python
from openai import OpenAI

client = OpenAI(base_url='http://localhost:3000/v1', api_key='na')

# Use the following func to get the available models
client.models.list()

chat_completion = client.chat.completions.create(
    model="mistralai/Mistral-7B-Instruct-v0.2",
    messages=[
        {
            "role": "user",
            "content": "Explain superconductors like I'm five years old"
        }
    ],
    stream=True,
)
for chunk in chat_completion:
    # Extract and print the content of the model's reply
    print(chunk.choices[0].delta.content or "", end="")
```

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

Make sure you have [logged in to BentoCloud](https://docs.bentoml.com/en/latest/bentocloud/how-tos/manage-access-token.html).

```bash
bentoml cloud login
```

Create a BentoCloud secret to store the required environment variable and reference it for deployment.

```bash
bentoml secret create huggingface HF_TOKEN=$HF_TOKEN

bentoml deploy . --secret huggingface
```

**Note**: For custom deployment in your own infrastructure, use [BentoML to generate an OCI-compliant image](https://docs.bentoml.com/en/latest/guides/containerization.html).

## Featured models

In addition to Mistral 7B Instruct, we also have examples for other models in the subdirectories of this repository. Here are some popular ones:

- [ai21-jamba-1.5-mini-int8](ai21-jamba-1.5-mini-int8/)
- [gemma-7b-it](gemma-7b-it/)
- [llama3.1-8b-instruct](llama3.1-8b-instruct/)
- [llama3.2-11b-instruct](llama3.2-11b-instruct/)
- [Mixtral-8x7B-Instruct-v0.1 with gptq quantization](mistral-7b-instruct/)
- [pixtral-12b](pixtral-12b/)
- [phi-3-mini-4k-instruct](phi-3-mini-4k-instruct/)
- [Outlines integration](outlines-integration/)
- [SOLAR-10.7B-v1.0](solar-10.7b-instruct/)