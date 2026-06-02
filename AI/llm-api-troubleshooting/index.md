# LLM API 调用问题诊断

调用 LLM provider(OpenAI / Azure / Vertex AI / Anthropic)时遇到的 timeout、状态码、fallback 设计相关知识汇总。来自一次 Azure Sweden Central + Vertex 同时抖动的实战诊断。

## 文件

- [[timeout-layers]] — Client / Gateway / Model 三层 timeout 的关系(理解 504 vs 499 的关键)
- [[error-status-codes]] — APITimeoutError / ReadTimeout / 499 / 504 / 429 / 400 各自含义和触发场景
- [[fallback-design]] — fallback 链怎么设计才有意义:跨 provider、跨 region、避免循环
- [[timeout-tuning]] — timeout 数值定多少合理:静态 p99 vs 动态按 token 数

## 一句话总结

> Provider 真挂了的时候,timeout 设多长都救不回来——能救你的是**跨 provider 的 fallback**。timeout 本身的作用只是**决定用户多快进入 fallback**,不是决定"模型能不能完成"。

## 常见误区(快速对照)

| 误区 | 真相 |
|---|---|
| "请求 timeout 了,说明 timeout 太短" | 看错误是否**集中在某时段**——集中=provider 抖动(加 timeout 没用),均匀分布=才考虑加 |
| "504 是因为我们 client timeout 太短" | 反了——504 是 **Vertex 自己**嫌 model 慢,我们 client 改 timeout 影响不到 |
| "fallback 配上就万事大吉" | 同 provider/region 的 fallback 在区域故障时**形同虚设** |
| "上下文越长越容易 timeout" | 是趋势,但具体诊断要看 token 数证据——10K-30K 都挂时,问题不在长度 |
