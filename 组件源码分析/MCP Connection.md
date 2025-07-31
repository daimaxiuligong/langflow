```go
import re
import shutil
from typing import Any
from langchain_core.tools import StructuredTool

# 导入 MCP 相关工具类和辅助函数
from langflow.base.mcp.util import (
    MCPSseClient,  # SSE 客户端,用于处理服务器发送事件连接
    MCPStdioClient,  # 标准输入输出客户端,用于命令行方式连接
    create_input_schema_from_json_schema,  # 将 JSON Schema 转换为输入模式的工具函数
    create_tool_coroutine,  # 创建异步工具协程的函数
    create_tool_func,  # 创建同步工具函数的函数
)

# 导入组件和输入/输出相关类
from langflow.custom import Component  # 基础组件类
from langflow.inputs import DropdownInput, TableInput  # 下拉菜单和表格输入组件
from langflow.inputs.inputs import InputTypes  # 输入类型定义
from langflow.io import MessageTextInput, MultilineInput, Output, TabInput  # 各种输入输出组件
from langflow.io.schema import flatten_schema, schema_to_langflow_inputs  # Schema 处理工具
from langflow.logging import logger  # 日志记录器
from langflow.schema import Message  # 消息模型

def maybe_unflatten_dict(flat: dict[str, Any]) -> dict[str, Any]:
    """
    将扁平化的字典转换回嵌套结构
    
    Args:
        flat (dict[str, Any]): 扁平化的字典,键可能包含点号(.)或数组索引([index])
        
    Returns:
        dict[str, Any]: 重建后的嵌套字典结构
        
    示例:
        输入: {"a.b": 1, "a.c[0]": 2, "a.c[1]": 3}
        输出: {"a": {"b": 1, "c": [2, 3]}}
    """
    # 快速检查是否存在需要展开的嵌套键
    if not any(re.search(r"\.|\[\d+\]", key) for key in flat):
        return flat

    # 初始化结果字典和数组正则表达式
    nested: dict[str, Any] = {}
    array_re = re.compile(r"^(.+)\[(\d+)\]$")

    # 遍历所有键值对进行重构
    for key, val in flat.items():
        parts = key.split(".")
        cur = nested
        for i, part in enumerate(parts):
            # 检查是否是数组索引格式
            m = array_re.match(part)
            if m:
                # 处理数组类型
                name, idx = m.group(1), int(m.group(2))
                lst = cur.setdefault(name, [])
                # 确保列表长度足够
                while len(lst) <= idx:
                    lst.append({})
                if i == len(parts) - 1:
                    lst[idx] = val  # 设置最终值
                else:
                    cur = lst[idx]  # 继续遍历
            else:
                # 处理普通对象属性
                if i == len(parts) - 1:
                    cur[part] = val  # 设置最终值
                else:
                    cur = cur.setdefault(part, {})  # 继续遍历

    return nested

class MCPToolsComponent(Component):
    """
    MCP 工具组件类
    
    用于连接 MCP 服务器并管理其提供的工具。支持两种连接模式:
    1. Stdio: 通过命令行方式连接
    2. SSE: 通过服务器发送事件方式连接
    
    主要功能:
    - 连接管理
    - 工具加载和缓存
    - 工具执行
    - 配置管理
    """
    
    # 基础属性定义
    schema_inputs: list[InputTypes] = []  # 模式输入列表
    stdio_client: MCPStdioClient = MCPStdioClient()  # Stdio 客户端实例
    sse_client: MCPSseClient = MCPSseClient()  # SSE 客户端实例
    tools: list = []  # 可用工具列表
    tool_names: list[str] = []  # 工具名称列表
    _tool_cache: dict = {}  # 工具对象缓存字典
    
    # 默认配置键列表
    default_keys: list[str] = [
        "code",  # 代码
        "_type",  # 类型
        "mode",  # 连接模式
        "command",  # 命令
        "env",  # 环境变量
        "sse_url",  # SSE URL
        "tool_placeholder",  # 工具占位符
        "tool_mode",  # 工具模式
        "tool",  # 工具
        "headers_input",  # 请求头
    ]

    # 组件显示相关属性
    display_name = "MCP Connection"  # 显示名称
    description = "Connect to an MCP server to use its tools."  # 描述
    icon = "Mcp"  # 图标
    name = "MCPTools"  # 组件名称

    # 定义组件输入项
    inputs = [
        TabInput(
            name="mode",
            display_name="Mode",
            options=["Stdio", "SSE"],
            value="Stdio",
            info="Select the connection mode",
            real_time_refresh=True,  # 实时刷新
        ),
        MessageTextInput(
            name="command",
            display_name="MCP Command",
            info="Command for MCP stdio connection",
            value="uvx mcp-server-fetch",
            show=True,
            refresh_button=True,  # 带刷新按钮
        ),
        MessageTextInput(
            name="env",
            display_name="Env",
            info="Env vars to include in mcp stdio connection (i.e. DEBUG=true)",
            value="",
            is_list=True,
            show=True,
            tool_mode=False,
            advanced=True,  # 高级选项
        ),
        MultilineInput(
            name="sse_url",
            display_name="MCP SSE URL",
            info="URL for MCP SSE connection",
            show=False,
            refresh_button=True,
            value="MCP_SSE",
            real_time_refresh=True,
        ),
        TableInput(
            name="headers_input",
            display_name="Headers",
            info="Headers to include in the tool",
            show=False,
            real_time_refresh=True,
            table_schema=[
                {
                    "name": "key",
                    "display_name": "Header",
                    "type": "str",
                    "description": "Header name",
                },
                {
                    "name": "value",
                    "display_name": "Value", 
                    "type": "str",
                    "description": "Header value",
                },
            ],
            value=[],
            advanced=True,
        ),
        DropdownInput(
            name="tool",
            display_name="Tool",
            options=[],
            value="",
            info="Select the tool to execute",
            show=True,
            required=True,
            real_time_refresh=True,
        ),
    ]

    # 定义组件输出
    outputs = [
        Output(
            display_name="Response",
            name="response",
            method="build_output",  # 指定输出构建方法
        ),
    ]

    async def _validate_connection_params(self, mode: str, command: str | None = None, url: str | None = None) -> None:
        """
        验证连接参数的有效性
        
        Args:
            mode (str): 连接模式 ('Stdio' 或 'SSE')
            command (str | None): Stdio 模式下的命令
            url (str | None): SSE 模式下的 URL
            
        Raises:
            ValueError: 当参数无效时抛出
        """
        if mode not in ["Stdio", "SSE"]:
            msg = f"Invalid mode: {mode}. Must be either 'Stdio' or 'SSE'"
            raise ValueError(msg)

        if mode == "Stdio" and not command:
            msg = "Command is required for Stdio mode"
            raise ValueError(msg)
        if mode == "Stdio" and command:
            self._validate_node_installation(command)
        if mode == "SSE" and not url:
            msg = "URL is required for SSE mode"
            raise ValueError(msg)

    def _validate_node_installation(self, command: str) -> str:
        """
        验证 Node.js 安装状态
        
        Args:
            command (str): 要执行的命令
            
        Returns:
            str: 验证通过的命令
            
        Raises:
            ValueError: 当 Node.js 未安装但命令需要时抛出
        """
        if "npx" in command and not shutil.which("node"):
            msg = "Node.js is not installed. Please install Node.js to use npx commands."
            raise ValueError(msg)
        return command

    def _process_headers(self, headers: Any) -> dict:
        """
        处理请求头数据为标准格式
        
        Args:
            headers (Any): 输入的头信息,可以是字典、字符串或列表
            
        Returns:
            dict: 处理后的请求头字典
        """
        if headers is None:
            return {}
        if isinstance(headers, dict):
            return headers
        if isinstance(headers, list):
            processed_headers = {}
            try:
                for item in headers:
                    if not self._is_valid_key_value_item(item):
                        continue
                    key = item["key"]
                    value = item["value"]
                    processed_headers[key] = value
            except (KeyError, TypeError, ValueError) as e:
                self.log(f"Failed to process headers list: {e}")
                return {}
            return processed_headers
        return {}

    def _is_valid_key_value_item(self, item: Any) -> bool:
        """
        检查项是否为有效的键值对字典
        
        Args:
            item (Any): 要检查的项
            
        Returns:
            bool: 如果是有效的键值对则返回 True
        """
        return isinstance(item, dict) and "key" in item and "value" in item

    async def _validate_schema_inputs(self, tool_obj) -> list[InputTypes]:
        """
        验证和处理工具的输入模式
        
        Args:
            tool_obj: 工具对象
            
        Returns:
            list[InputTypes]: 处理后的输入类型列表
            
        Raises:
            ValueError: 当验证失败时抛出
        """
        try:
            if not tool_obj or not hasattr(tool_obj, "inputSchema"):
                msg = "Invalid tool object or missing input schema"
                raise ValueError(msg)

            flat_schema = flatten_schema(tool_obj.inputSchema)
            input_schema = create_input_schema_from_json_schema(flat_schema)
            if not input_schema:
                msg = f"Empty input schema for tool '{tool_obj.name}'"
                raise ValueError(msg)

            schema_inputs = schema_to_langflow_inputs(input_schema)
            if not schema_inputs:
                msg = f"No input parameters defined for tool '{tool_obj.name}'"
                logger.warning(msg)
                return []

        except Exception as e:
            msg = f"Error validating schema inputs: {e!s}"
            logger.exception(msg)
            raise ValueError(msg) from e
        else:
            return schema_inputs

    async def update_build_config(self, build_config: dict, field_value: str, field_name: str | None = None) -> dict:
        """
        根据字段变化更新构建配置
        
        Args:
            build_config (dict): 当前构建配置
            field_value (str): 字段新值
            field_name (str | None): 变化的字段名
            
        Returns:
            dict: 更新后的构建配置
            
        Raises:
            ValueError: 更新过程中出错时抛出
        """
        try:
            # 处理模式切换
            if field_name == "mode":
                self.remove_non_default_keys(build_config)
                build_config["tool"]["options"] = []
                if field_value == "Stdio":
                    # 设置 Stdio 模式的显示配置
                    build_config["command"]["show"] = True
                    build_config["env"]["show"] = True
                    build_config["headers_input"]["show"] = False
                    build_config["sse_url"]["show"] = False
                elif field_value == "SSE":
                    # 设置 SSE 模式的显示配置
                    build_config["command"]["show"] = False
                    build_config["env"]["show"] = False
                    build_config["sse_url"]["show"] = True
                    build_config["sse_url"]["value"] = "MCP_SSE"
                    build_config["headers_input"]["show"] = True
                    return build_config
                    
            # 处理连接相关字段变化
            if field_name in ("command", "sse_url", "mode"):
                try:
                    await self.update_tools(
                        mode=build_config["mode"]["value"],
                        command=build_config["command"]["value"],
                        url=build_config["sse_url"]["value"],
                        env=build_config["env"]["value"],
                        headers=build_config["headers_input"]["value"],
                    )
                    if "tool" in build_config:
                        build_config["tool"]["options"] = self.tool_names
                except Exception as e:
                    build_config["tool"]["options"] = []
                    msg = f"Failed to update tools: {e!s}"
                    raise ValueError(msg) from e
                else:
                    return build_config
                    
            # 处理工具选择变化
            elif field_name == "tool":
                # 确保工具列表已更新
                if len(self.tools) == 0:
                    await self.update_tools(
                        mode=build_config["mode"]["value"],
                        command=build_config["command"]["value"],
                        url=build_config["sse_url"]["value"],
                        env=build_config["env"]["value"],
                        headers=build_config["headers_input"]["value"],
                    )
                    
                if self.tool is None:
                    return build_config
                    
                # 查找选中的工具对象
                tool_obj = None
                for tool in self.tools:
                    if tool.name == self.tool:
                        tool_obj = tool
                        break
                        
                if tool_obj is None:
                    msg = f"Tool {self.tool} not found in available tools: {self.tools}"
                    logger.warning(msg)
                    return build_config
                    
                # 更新工具配置
                self.remove_non_default_keys(build_config)
                await self._update_tool_config(build_config, field_value)
                
            # 处理工具模式变化
            elif field_name == "tool_mode":
                build_config["tool"]["show"] = not field_value
                for key, value in list(build_config.items()):
                    if key not in self.default_keys and isinstance(value, dict) and "show" in value:
                        build_config[key]["show"] = not field_value

        except Exception as e:
            msg = f"Error in update_build_config: {e!s}"
            logger.exception(msg)
            raise ValueError(msg) from e
        else:
            return build_config

    def get_inputs_for_all_tools(self, tools: list) -> dict:
        """
        获取所有工具的输入模式
        
        Args:
            tools (list): 工具列表
            
        Returns:
            dict: 工具名称到输入模式的映射字典
        """
        inputs = {}
        for tool in tools:
            if not tool or not hasattr(tool, "name"):
                continue
            try:
                flat_schema = flatten_schema(tool.inputSchema)
                input_schema = create_input_schema_from_json_schema(flat_schema)
                langflow_inputs = schema_to_langflow_inputs(input_schema)
                inputs[tool.name] = langflow_inputs
            except (AttributeError, ValueError, TypeError, KeyError) as e:
                msg = f"Error getting inputs for tool {getattr(tool, 'name', 'unknown')}: {e!s}"
                logger.exception(msg)
                continue
        return inputs

    def remove_input_schema_from_build_config(
        self, build_config: dict, tool_name: str, input_schema: dict[list[InputTypes], Any]
    ):
        """
        从构建配置中移除指定工具的输入模式
        
        Args:
            build_config (dict): 构建配置
            tool_name (str): 工具名称
            input_schema (dict): 输入模式字典
        """
        # 保留不属于当前工具的模式
        input_schema = {k: v for k, v in input_schema.items() if k != tool_name}
        # 移除其他工具的所有输入
        for value in input_schema.values():
            for _input in value:
                if _input.name in build_config:
                    build_config.pop(_input.name)

    def remove_non_default_keys(self, build_config: dict) -> None:
        """
        从构建配置中移除非默认键
        
        Args:
            build_config (dict): 构建配置
        """
        for key in list(build_config.keys()):
            if key not in self.default_keys:
                build_config.pop(key)

    async def _update_tool_config(self, build_config: dict, tool_name: str) -> None:
        """
        更新工具配置
        
        Args:
            build_config (dict): 构建配置
            tool_name (str): 工具名称
            
        Raises:
            ValueError: 配置更新失败时抛出
        """
        if not self.tools:
            await self.update_tools(
                mode=build_config["mode"]["value"],
                command=build_config["command"]["value"],
                url=build_config["sse_url"]["value"],
                env=build_config["env"]["value"],
                headers=build_config["headers_input"]["value"],
            )

        if not tool_name:
            return

        # 查找工具对象
        tool_obj = next((tool for tool in self.tools if tool.name == tool_name), None)
        if not tool_obj:
            msg = f"Tool {tool_name} not found in available tools: {self.tools}"
            logger.warning(msg)
            return

        try:
            # 获取所有工具输入并移除旧的
            input_schema_for_all_tools = self.get_inputs_for_all_tools(self.tools)
            self.remove_input_schema_from_build_config(build_config, tool_name, input_schema_for_all_tools)

            # 获取并验证新输入
            self.schema_inputs = await self._validate_schema_inputs(tool_obj)
            if not self.schema_inputs:
                msg = f"No input parameters to configure for tool '{tool_name}'"
                logger.info(msg)
                return

            # 将新输入添加到构建配置
            for schema_input in self.schema_inputs:
                if not schema_input or not hasattr(schema_input, "name"):
                    msg = "Invalid schema input detected, skipping"
                    logger.warning(msg)
                    continue

                try:
                    name = schema_input.name
                    input_dict = schema_input.to_dict()
                    input_dict.setdefault("value", None)
                    input_dict.setdefault("required", True)
                    build_config[name] = input_dict
                except (AttributeError, KeyError, TypeError) as e:
                    msg = f"Error processing schema input {schema_input}: {e!s}"
                    logger.exception(msg)
                    continue
        except ValueError as e:
            msg = f"Schema validation error for tool {tool_name}: {e!s}"
            logger.exception(msg)
            self.schema_inputs = []
            return
        except (AttributeError, KeyError, TypeError) as e:
            msg = f"Error updating tool config: {e!s}"
            logger.exception(msg)
            raise ValueError(msg) from e

    async def build_output(self) -> Message:
        """
        构建组件输出
        
        Returns:
            Message: 输出消息对象
            
        Raises:
            ValueError: 构建输出失败时抛出
        """
        try:
            await self.update_tools()
            if self.tool != "":
                # 获取工具和参数
                exec_tool = self._tool_cache[self.tool]
                tool_args = self.get_inputs_for_all_tools(self.tools)[self.tool]
                kwargs = {}
                for arg in tool_args:
                    value = getattr(self, arg.name, None)
                    if value:
                        kwargs[arg.name] = value

                # 展开参数并执行工具
                unflattened_kwargs = maybe_unflatten_dict(kwargs)
                output = await exec_tool.coroutine(**unflattened_kwargs)

                return Message(text=output.content[len(output.content) - 1].text)
            return Message(text="You must select a tool", error=True)
        except Exception as e:
            msg = f"Error in build_output: {e!s}"
            logger.exception(msg)
            raise ValueError(msg) from e

    async def update_tools(
        self,
        mode: str | None = None,
        command: str | None = None,
        url: str | None = None,
        env: list[str] | None = None,
        headers: dict[str, str] | None = None,
    ) -> list[StructuredTool]:
        """
        连接到 MCP 服务器并更新可用工具列表
        
        Args:
            mode (str | None): 连接模式
            command (str | None): Stdio 模式的命令
            url (str | None): SSE 模式的 URL
            env (list[str] | None): 环境变量
            headers (dict[str, str] | None): 请求头
            
        Returns:
            list[StructuredTool]: 工具列表
            
        Raises:
            ValueError: 更新失败时抛出
        """
        try:
            # 使用默认值
            if mode is None:
                mode = self.mode
            if command is None:
                command = self.command
            if env is None:
                env = self.env
            if url is None:
                url = self.sse_url
            if headers is None:
                headers = self.headers_input
                
            # 处理请求头
            headers = self._process_headers(headers)
            await self._validate_connection_params(mode, command, url)

            # 根据模式建立连接
            if mode == "Stdio":
                if not self.stdio_client.session:
                    try:
                        self.tools = await self.stdio_client.connect_to_server(command, env)
                    except ValueError as e:
                        msg = f"Error connecting to MCP server: {e}"
                        logger.exception(msg)
                        raise ValueError(msg) from e
            elif mode == "SSE" and not self.sse_client.session:
                try:
                    self.tools = await self.sse_client.connect_to_server(url, headers)
                except ValueError as e:
                    # URL 验证错误
                    logger.error(f"SSE URL validation error: {e}")
                    msg = f"Invalid SSE URL configuration: {e}. Please check your Langflow deployment URL and port."
                    raise ValueError(msg) from e
                except ConnectionError as e:
                    # 连接重试后失败
                    logger.error(f"SSE connection error: {e}")
                    msg = (
                        f"Could not connect to Langflow SSE endpoint: {e}. "
                        "Please verify:\n"
                        "1. Langflow server is running\n"
                        "2. The SSE URL matches your Langflow deployment port\n"
                        "3. There are no network issues preventing the connection"
                    )
                    raise ValueError(msg) from e
                except Exception as e:
                    logger.error(f"Unexpected SSE error: {e}")
                    msg = f"Unexpected error connecting to SSE endpoint: {e}"
                    raise ValueError(msg) from e

            if not self.tools:
                logger.warning("No tools returned from server")
                return []

            # 处理工具列表
            tool_list = []
            for tool in self.tools:
                if not tool or not hasattr(tool, "name"):
                    logger.warning("Invalid tool object detected, skipping")
                    continue

                try:
                    # 创建工具参数模式
                    args_schema = create_input_schema_from_json_schema(tool.inputSchema)
                    if not args_schema:
                        logger.warning(f"Empty schema for tool '{tool.name}', skipping")
                        continue

                    # 验证客户端会话
                    client = self.stdio_client if self.mode == "Stdio" else self.sse_client
                    if not client or not client.session:
                        msg = f"Invalid client session for tool '{tool.name}'"
                        raise ValueError(msg)

                    # 创建工具对象
                    tool_obj = StructuredTool(
                        name=tool.name,
                        description=tool.description or "",
                        args_schema=args_schema,
                        func=create_tool_func(tool.name, args_schema, client.session),
                        coroutine=create_tool_coroutine(tool.name, args_schema, client.session),
                        tags=[tool.name],
                        metadata={},
                    )
                    tool_list.append(tool_obj)
                    self._tool_cache[tool.name] = tool_obj
                except (AttributeError, ValueError, TypeError, KeyError) as e:
                    msg = f"Error creating tool {getattr(tool, 'name', 'unknown')}: {e}"
                    logger.exception(msg)
                    continue

            # 更新工具名称列表
            self.tool_names = [tool.name for tool in self.tools if hasattr(tool, "name")]

        except ValueError as e:
            # 重新抛出验证错误,保持明确的错误消息
            raise ValueError(str(e)) from e
        except Exception as e:
            logger.exception("Error updating tools")
            msg = f"Failed to update tools: {e!s}"
            raise ValueError(msg) from e
        else:
            return tool_list

    async def _get_tools(self):
        """
        获取缓存的工具或必要时更新
        
        Returns:
            list: 工具列表
            
        Raises:
            ValueError: 获取失败时抛出
        """
        if self.mode == "SSE" and self.sse_url is None:
            msg = "SSE URL is not set"
            raise ValueError(msg)
        return await self.update_tools()
```

