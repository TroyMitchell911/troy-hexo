title: i2c-adpater-mainline for k1 of spacemit
date: '2024-07-17 13:03:47'
updated: '2024-07-17 13:03:53'
tags:
  - kernel
  - linux
categories:
  - kernel
---
## 初步完善框架

```c
#include <linux/mod_devicetable.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/device.h>
#include <linux/i2c.h>
#include <linux/err.h>
#include <linux/clk.h>

#include "i2c-k1x.h"

static int
spacemit_k1_i2c_xfer(struct i2c_adapter *adapt, struct i2c_msg msgs[], int num) {
	struct spacemit_k1_i2c *i2c = i2c_get_adapdata(adapt);

	return num;
}

static u32 spacemit_k1_i2c_functionality(struct i2c_adapter *adap) {
	return I2C_FUNC_I2C | (I2C_FUNC_SMBUS_EMUL & ~I2C_FUNC_SMBUS_QUICK);
}

static const struct i2c_algorithm spacemit_k1_i2c_algrtm = {
	.master_xfer	= spacemit_k1_i2c_xfer,
	.functionality	= spacemit_k1_i2c_functionality,
};


static int spacemit_k1_i2c_parse_dt(struct platform_device *pdev,
				    struct spacemit_k1_i2c *i2c) {
	struct device_node *dnode = pdev->dev.of_node;
	int ret;

	ret = of_property_read_u32(dnode, "spacemit,adapter-id", &pdev->id);
	if(ret)
		pdev->id = -1;
	
	return 0;
}

static int spacemit_k1_i2c_probe(struct platform_device *pdev) {
	struct spacemit_k1_i2c *i2c;
	int ret = 0;

	i2c = devm_kzalloc(&pdev->dev,
			sizeof(struct spacemit_k1_i2c),
			GFP_KERNEL);
	
	if (!i2c) {
		ret = -ENOMEM;	
		goto err_ret;
	}

	i2c->dev = &pdev->dev;
	platform_set_drvdata(pdev, i2c);
	
	ret = spacemit_k1_i2c_parse_dt(pdev, i2c);
	if(ret)
		goto err_ret;	
	
	pr_info("spacemit_k1_i2c_probe function: pdev->id %d\n", pdev->id);

	i2c->adap.owner 	= THIS_MODULE;
	i2c->adap.algo 		= &spacemit_k1_i2c_algrtm;
	i2c->adap.algo_data 	= i2c;
	i2c->adap.nr 		= pdev->id;
	i2c->adap.dev.parent 	= &pdev->dev;
	i2c->adap.dev.of_node 	= pdev->dev.of_node;
	i2c->adap.retries	= 3;
	strscpy(i2c->adap.name, "spacemit-i2c-adapter",
		sizeof(i2c->adap.name));
	ret = i2c_add_numbered_adapter(&i2c->adap);
	if(ret) {
		dev_err(i2c->dev, "failed to add i2c adapter\n");
		goto err_ret;
	}	

err_ret:
	return ret;
}

static int spacemit_k1_i2c_remove(struct platform_device *pdev) {
	struct spacemit_k1_i2c *i2c;
	
	pr_info("spacemit_k1_i2c_remove\n");
	
	i2c = platform_get_drvdata(pdev);

	i2c_del_adapter(&i2c->adap);

	dev_dbg(i2c->dev, "driver removed\n");

	return 0;
}

static const struct of_device_id spacemit_k1_i2c_dt_match[] = {
	{
		.compatible = "spacemit,k1x-i2c",
	},
	{}
};

MODULE_DEVICE_TABLE(of, spacemit_k1_i2c_dt_match);

static struct platform_driver spacemit_k1_i2c_driver = {
	.probe  = spacemit_k1_i2c_probe,
	.remove = spacemit_k1_i2c_remove,
	.driver = {
		.name		= "i2c-spacemit-k1x",
		.of_match_table	= spacemit_k1_i2c_dt_match,
	},
};


module_platform_driver(spacemit_k1_i2c_driver);
MODULE_LICENSE("GPL v2");
MODULE_DESCRIPTION("i2c driver for Spacemit k1 soc");
```

这里选择`module_platform_driver`而不是像`vendor`版本的写法是因为我们暂时不需要注册重启`hook`。

k1写法如下：

```c
static int __init spacemit_i2c_init(void)
{
	register_restart_handler(&spacemit_i2c_sys_nb);
	return platform_driver_register(&spacemit_i2c_driver);
}

static void __exit spacemit_i2c_exit(void)
{
	platform_driver_unregister(&spacemit_i2c_driver);
	unregister_restart_handler(&spacemit_i2c_sys_nb);
}

subsys_initcall(spacemit_i2c_init);
module_exit(spacemit_i2c_exit);
```