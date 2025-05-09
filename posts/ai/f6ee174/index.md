# MCP和LLM的调用细节


{{&lt; admonition type=info title=&#34;内容简介&#34; open=true &gt;}}
介绍MCP和LLM之间的协作细节，讲解LLM是如何利用MCP服务来扩展自己的边际能力的。
{{&lt; /admonition &gt;}}

在用go做了几个MCP的Demo服务后，又对MCP的架构和协议细节进行了深入地学习，对MCP的理解深刻了很多。但是在开发过程中还是有两个关键的问题未得到解答：

- cline插件、我的MCP服务、大模型这三者之间的调用流程是怎样的？
- 大模型是在什么时候确定使用哪些MCP服务的呢？

这里需要注意下，我使用的是vscode的cline插件，所以这里拿cline举例，但是其实客户端也可以是cursor、cherry studio等其他客户端。

在 MCP 官网为我们提供了一个[解释](https://modelcontextprotocol.io/quickstart/server#what%E2%80%99s-happening-under-the-hood)：
1. 客户端将你的问题发送给 Claude
2. Claude 分析可用的工具，并决定使用哪一个或多个
3. 客户端通过 MCP Server 执行所选的工具
4. 工具的执行结果被送回给 Claude
5. Claude 结合执行结果生成回答
6. 回应最终展示给用户

从以上的解释可以看出，大模型和MCP服务之间的调用过程是分两步完成的：
1. 由 LLM 确定使用哪些 MCP Server
2. 执行对应的 MCP Server 并对执行结果进行重新处理

所以 MCP Server 是由大模型主动选择并调用的。但是大模型具体又是如何确定该使用哪些工具呢？从 MCP 官方提供的 [pyhton client example](https://github.com/modelcontextprotocol/python-sdk/blob/main/examples/clients/simple-chatbot/mcp_simple_chatbot/main.py) 中可以得到答案：
```python
    async def start(self) -&gt; None:
        &#34;&#34;&#34;Main chat session handler.&#34;&#34;&#34;
        try:
            for server in self.servers:
                try:
                    await server.initialize()
                except Exception as e:
                    logging.error(f&#34;Failed to initialize server: {e}&#34;)
                    await self.cleanup_servers()
                    return

            all_tools = []
            for server in self.servers:
                tools = await server.list_tools()
                all_tools.extend(tools)

            tools_description = &#34;\n&#34;.join([tool.format_for_llm() for tool in all_tools])

            system_message = (
                &#34;You are a helpful assistant with access to these tools:\n\n&#34;
                f&#34;{tools_description}\n&#34;
                &#34;Choose the appropriate tool based on the user&#39;s question. &#34;
                &#34;If no tool is needed, reply directly.\n\n&#34;
                &#34;IMPORTANT: When you need to use a tool, you must ONLY respond with &#34;
                &#34;the exact JSON object format below, nothing else:\n&#34;
                &#34;{\n&#34;
                &#39;    &#34;tool&#34;: &#34;tool-name&#34;,\n&#39;
                &#39;    &#34;arguments&#34;: {\n&#39;
                &#39;        &#34;argument-name&#34;: &#34;value&#34;\n&#39;
                &#34;    }\n&#34;
                &#34;}\n\n&#34;
                &#34;After receiving a tool&#39;s response:\n&#34;
                &#34;1. Transform the raw data into a natural, conversational response\n&#34;
                &#34;2. Keep responses concise but informative\n&#34;
                &#34;3. Focus on the most relevant information\n&#34;
                &#34;4. Use appropriate context from the user&#39;s question\n&#34;
                &#34;5. Avoid simply repeating the raw data\n\n&#34;
                &#34;Please use only the tools that are explicitly defined above.&#34;
            )

            messages = [{&#34;role&#34;: &#34;system&#34;, &#34;content&#34;: system_message}]

            while True:
                try:
                    user_input = input(&#34;You: &#34;).strip().lower()
                    if user_input in [&#34;quit&#34;, &#34;exit&#34;]:
                        logging.info(&#34;\nExiting...&#34;)
                        break

                    messages.append({&#34;role&#34;: &#34;user&#34;, &#34;content&#34;: user_input})

                    llm_response = self.llm_client.get_response(messages)
                    logging.info(&#34;\nAssistant: %s&#34;, llm_response)

                    result = await self.process_llm_response(llm_response)

                    if result != llm_response:
                        messages.append({&#34;role&#34;: &#34;assistant&#34;, &#34;content&#34;: llm_response})
                        messages.append({&#34;role&#34;: &#34;system&#34;, &#34;content&#34;: result})

                        final_response = self.llm_client.get_response(messages)
                        logging.info(&#34;\nFinal response: %s&#34;, final_response)
                        messages.append(
                            {&#34;role&#34;: &#34;assistant&#34;, &#34;content&#34;: final_response}
                        )
                    else:
                        messages.append({&#34;role&#34;: &#34;assistant&#34;, &#34;content&#34;: llm_response})

                except KeyboardInterrupt:
                    logging.info(&#34;\nExiting...&#34;)
                    break

        finally:
            await self.cleanup_servers()

            ... # 省略其他代码
```
从代码可以看出，在和大模型进行交互前，将所有工具的结构化描述放到`tools_description`中，再添加到`system_message`中，然后把`system_message`和用户消息一起发送给模型。当模型分析用户请求后，它会决定是否需要调用工具：
- 无需工具时：模型直接生成自然语言回复。
- 需要工具时：模型输出结构化 JSON 格式的工具调用请求。

当回复中包含结构化 JSON 格式的工具调用请求，则客户端会根据这个 json 代码调用对应的工具。以上两点可以在`get_response`、`process_llm_response`中看到实现，代码逻辑非常简单：
```python
    def get_response(self, messages: list[dict[str, str]]) -&gt; str:
        &#34;&#34;&#34;Get a response from the LLM.

        Args:
            messages: A list of message dictionaries.

        Returns:
            The LLM&#39;s response as a string.

        Raises:
            httpx.RequestError: If the request to the LLM fails.
        &#34;&#34;&#34;
        url = &#34;https://api.groq.com/openai/v1/chat/completions&#34;

        headers = {
            &#34;Content-Type&#34;: &#34;application/json&#34;,
            &#34;Authorization&#34;: f&#34;Bearer {self.api_key}&#34;,
        }
        payload = {
            &#34;messages&#34;: messages,
            &#34;model&#34;: &#34;llama-3.2-90b-vision-preview&#34;,
            &#34;temperature&#34;: 0.7,
            &#34;max_tokens&#34;: 4096,
            &#34;top_p&#34;: 1,
            &#34;stream&#34;: False,
            &#34;stop&#34;: None,
        }

        try:
            with httpx.Client() as client:
                response = client.post(url, headers=headers, json=payload)
                response.raise_for_status()
                data = response.json()
                return data[&#34;choices&#34;][0][&#34;message&#34;][&#34;content&#34;]

        except httpx.RequestError as e:
            error_message = f&#34;Error getting LLM response: {str(e)}&#34;
            logging.error(error_message)

            if isinstance(e, httpx.HTTPStatusError):
                status_code = e.response.status_code
                logging.error(f&#34;Status code: {status_code}&#34;)
                logging.error(f&#34;Response details: {e.response.text}&#34;)

            return (
                f&#34;I encountered an error: {error_message}. &#34;
                &#34;Please try again or rephrase your request.&#34;
            )

    # 省略部分代码...

    async def process_llm_response(self, llm_response: str) -&gt; str:
        &#34;&#34;&#34;Process the LLM response and execute tools if needed.

        Args:
            llm_response: The response from the LLM.

        Returns:
            The result of tool execution or the original response.
        &#34;&#34;&#34;
        import json

        try:
            tool_call = json.loads(llm_response)
            if &#34;tool&#34; in tool_call and &#34;arguments&#34; in tool_call:
                logging.info(f&#34;Executing tool: {tool_call[&#39;tool&#39;]}&#34;)
                logging.info(f&#34;With arguments: {tool_call[&#39;arguments&#39;]}&#34;)

                for server in self.servers:
                    tools = await server.list_tools()
                    if any(tool.name == tool_call[&#34;tool&#34;] for tool in tools):
                        try:
                            result = await server.execute_tool(
                                tool_call[&#34;tool&#34;], tool_call[&#34;arguments&#34;]
                            )

                            if isinstance(result, dict) and &#34;progress&#34; in result:
                                progress = result[&#34;progress&#34;]
                                total = result[&#34;total&#34;]
                                percentage = (progress / total) * 100
                                logging.info(
                                    f&#34;Progress: {progress}/{total} &#34;
                                    f&#34;({percentage:.1f}%)&#34;
                                )

                            return f&#34;Tool execution result: {result}&#34;
                        except Exception as e:
                            error_msg = f&#34;Error executing tool: {str(e)}&#34;
                            logging.error(error_msg)
                            return error_msg

                return f&#34;No server found with tool: {tool_call[&#39;tool&#39;]}&#34;
            return llm_response
        except json.JSONDecodeError:
            return llm_response
```
至此，MCP服务和LLM之间的调用过程已经很明朗了：
{{&lt; figure src=&#34;https://images.benelin.site/posts/mcp-process.jpg&#34; width=800 height=800 &gt;}}

依据以上实现原理可以总结：
- **MCP 的出现是 prompt engineering 发展的产物。模型是通过 prompt engineering，即提供所有工具的结构化描述来确定该使用哪些工具的。**
- **MCP 工具文档至关重要，模型通过工具描述文本来理解和选择工具，因此精心编写工具的名称、docstring 和参数说明至关重要。**
- **由于 MCP 的选择是基于 prompt 的，所以任何模型其实都适配 MCP。**

**参考资料**
1. [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk/blob/main/examples/clients/simple-chatbot/mcp_simple_chatbot/main.py)
2. [如何用go搭建MCP服务](https://mp.weixin.qq.com/s/6j3q4LAOvCgUo54xE_Y5Vg)





---

> 作者: [beneliu](https://github.com/beneliu)  
> URL: https://beneliu.github.io/posts/ai/f6ee174/  

