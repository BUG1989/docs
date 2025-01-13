---
sidebar_position: 1
description: "Linux 下通过 USB 刷机"
---

import Rkdeveloptool from "../../../../common/dev/\_rkdeveloptoolV2.mdx";
import Images from "../../\_image.mdx"

# Linux 主机

## 文件下载

<Images loader={true} rock5t_system_img_61={true} spi_img={false} />

<Rkdeveloptool model="rock-5t" release_num="t1" desktop="kde" platform="linux" loader="rk3588_spl_loader_v1.08.111.bin">

<TabItem value="ROCK 5T">
        <ol>
            <li>拔出 microSD 卡和电源线</li>
            <li>按住 Maskrom button</li>
            <li>将 USB A-C 电缆插入 ROCK 5T 的 Type-C 端口，另一端插入电脑，给 ROCK 5T 上电，此时正常情况下会进入 Maskrom 状态</li>
        </ol>
        <img src="/img/rock5t/rock-5t-typec-maskrom.webp" alt="rock 5t maskrom wire" width="500" />
    </TabItem>

</Rkdeveloptool>
