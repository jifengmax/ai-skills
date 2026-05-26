# AI Skills（task-delegation-orchestrator）

My personal collection of [Hermes Agent](https://hermes-agent.nousresearch.com) skills.中国人开发的，相当于设置一个项目经理，你把需求对项目经理说，他会安排对项目或者任务进行分解，然后逐项执行，最后他审验，有问题的就打回去重做，最后给你交付一个比较严谨的结果。运用的harness技术，和子代理委派技术，如果你有多个子agents，可以作为一个团队一起工作，效果出奇的好。命中率也很高，因为是做了任务分解，所以每个子agent不会完整的读取以前的历史对话，能节省token，并且几个代理同时干，速度和效率都倍增。你的AI Agent装好它之后，一定要让它在每次你派发任务时候，问你一句，是否启用子代理委派模式，如果启用他就会调用此skill。简单的任务就不许要了。

## Skills

- **[task-delegation-orchestrator](software-development/task-delegation-orchestrator/SKILL.md)** — Multi-agent task orchestration: decompose, dispatch, review, rework, and deliver.

## Usage

You can load any skill via `hermes skills install https://raw.githubusercontent.com/jifengmax/ai-skills/main/{path_to_skill}/SKILL.md`
