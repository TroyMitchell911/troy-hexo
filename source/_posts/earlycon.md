title: earlycon
date: '2024-07-03 18:01:54'
updated: '2024-07-29 16:24:19'
tags:
  - kernel
  - serial
  - linux
categories:
  - kernel
---
## 引言

`earlycon` 是一个早期控制台（early console）机制，用于在系统启动的早期阶段提供输出功能。在内核启动过程的早期阶段，标准的控制台设备（如串口、VGA控制台等）可能还没有初始化完成，这时可以使用 `earlycon` 来输出调试信息，帮助开发者调试内核启动过程中的问题。

## 如何开启earlycon

要在内核启动时启用 `earlycon`，需要在内核配置中启用几个相关的配置选项：

```
CONFIG_SERIAL_EARLYCON
CONFIG_OF_EARLY_FLATTREE
```

还需要在内核命令行参数中添加相关设置。例如：

```
earlycon=pxa_serial,0xd4017000
```

## 具体流程

在 `Kernel` 初始化汇编代码执行完跳转到 `start_kernel` 之后，`setup_arch` 调用 `parse_early_param`，进而在其中执行 `early_param` 的解析，具体如下：

start_kernel->setup_arch->parse_early_param->parse_early_options->do_early_param

```c
// In init/main.c
void start_kernel(void) {
    char *command_line;
    ...
	setup_arch(&command_line);
    ...
}
```

```c
// In arch/riscv/kernel/setup.c
void __init setup_arch(char **cmdline_p)
{
    ...
	parse_early_param();
    ...
```

```c
// In arch/riscv/kernel/setup.c
void __init parse_early_param(void)
{
	static int done __initdata;
	static char tmp_cmdline[COMMAND_LINE_SIZE] __initdata;

	if (done)
		return;

	/* All fall through to do_early_param. */
	strscpy(tmp_cmdline, boot_command_line, COMMAND_LINE_SIZE);
	parse_early_options(tmp_cmdline);
	done = 1;
}
```

```c
// In arch/riscv/kernel/setup.c
void __init parse_early_options(char *cmdline)
{
	parse_args("early options", cmdline, NULL, 0, 0, 0, NULL,
		   do_early_param);
}
```

从这里看到有一个`parse_args`函数，其函数目的是`解析键值`对参数并将键值对作为参数`执行`最后一个参数写入的函数指针。

```c
// In kernel/params.c
char *parse_args(const char *doing,
		 char *args,
		 const struct kernel_param *params,
		 unsigned num,
		 s16 min_level,
		 s16 max_level,
		 void *arg, parse_unknown_fn unknown)
{
	char *param, *val, *err = NULL;

	/* Chew leading spaces */
	args = skip_spaces(args);

	if (*args)
		pr_debug("doing %s, parsing ARGS: '%s'\n", doing, args);

	while (*args) {
		int ret;
		int irq_was_disabled;

		args = next_arg(args, &param, &val);
		/* Stop at -- */
		if (!val && strcmp(param, "--") == 0)
			return err ?: args;
		irq_was_disabled = irqs_disabled();
		ret = parse_one(param, val, doing, params, num,
				min_level, max_level, arg, unknown);
		if (irq_was_disabled && !irqs_disabled())
			pr_warn("%s: option '%s' enabled irq's!\n",
				doing, param);

		switch (ret) {
		case 0:
			continue;
		case -ENOENT:
			pr_err("%s: Unknown parameter `%s'\n", doing, param);
			break;
		case -ENOSPC:
			pr_err("%s: `%s' too large for parameter `%s'\n",
			       doing, val ?: "", param);
			break;
		default:
			pr_err("%s: `%s' invalid for parameter `%s'\n",
			       doing, val ?: "", param);
			break;
		}

		err = ERR_PTR(ret);
	}

	return err;
}

```

```c
// In arch/riscv/kernel/setup.c
static int __init do_early_param(char *param, char *val,
				 const char *unused, void *arg)
{
	const struct obs_kernel_param *p;

	for (p = __setup_start; p < __setup_end; p++) {
		if ((p->early && parameq(param, p->str)) ||
		    (strcmp(param, "console") == 0 &&
		     strcmp(p->str, "earlycon") == 0)
		) {
			if (p->setup_func(val) != 0)
				pr_warn("Malformed early option '%s'\n", param);
		}
	}
	/* We accept everything at this stage. */
	return 0;
}
```

`__setup_start`和`__setup_end`变量在`arch/riscv/kernel/vmlinux.lds`中可以查找到相关定义，他们是`.init.setup`段的开始地址和结束地址：
```
.init.data : AT(ADDR(.init.data) - ((((-1))) - 0x80000000 + 1)) {
    ... 
    __setup_start = .; KEEP(*(.init.setup)) __setup_end = .;  
    ...
}
```

可以看到`__setup_start`和`__setup_end`都属于`.init.setup`段，经过搜索在`include/linux/init.h`发现有宏定义可以将函数放入该段：
```c
/*
 * Only for really core code.  See moduleparam.h for the normal way.
 *
 * Force the alignment so the compiler doesn't space elements of the
 * obs_kernel_param "array" too far apart in .init.setup.
 */
#define __setup_param(str, unique_id, fn, early)			\
	static const char __setup_str_##unique_id[] __initconst		\
		__aligned(1) = str; 					\
	static struct obs_kernel_param __setup_##unique_id		\
		__used __section(".init.setup")				\
		__aligned(__alignof__(struct obs_kernel_param))		\
		= { __setup_str_##unique_id, fn, early }

/*
 * NOTE: __setup functions return values:
 * @fn returns 1 (or non-zero) if the option argument is "handled"
 * and returns 0 if the option argument is "not handled".
 */
#define __setup(str, fn)						\
	__setup_param(str, fn, fn, 0)

/*
 * NOTE: @fn is as per module_param, not __setup!
 * I.e., @fn returns 0 for no error or non-zero for error
 * (possibly @fn returns a -errno value, but it does not matter).
 * Emits warning if @fn returns non-zero.
 */
#define early_param(str, fn)						\
	__setup_param(str, fn, fn, 1)
```

通过以上代码不难看出，`__setup_param`宏定义被`__setup`和`early_param`两个宏调用，具体区别就是`early`宏参数是`0`还是`1`。

在`do_early_param`函数中，如果`early`不是`1`的话就不会在`early`阶段解析，具体在哪个阶段不是本章的范畴，那么看来我们只关心`early_param`这个宏定义即可。

`e.g. early_param("earlycon", earlycon_func)` 

将`early_param`一步步展开得到如下结论：
```c
    early_param("earlycon", earlycon_func) 
    ->
	__setup_param("earlycon", earlycon_func, earlycon_func, 1)  
    ->
    	static const char __setup_str_earlycon_func[] __initconst		\
		__aligned(1) = "earlycon"; 					\
	static struct obs_kernel_param __setup_earlycon_func		\
		__used __section(".init.setup")				\
		__aligned(__alignof__(struct obs_kernel_param))		\
		= { __setup_str_earlycon_func, earlycon_func, 1 }
```
经过上面推导展开不难发现，其本质就是声明了一个结构体，结构体类型是`struct obs_kernel_param`，并且在该结构体定义的地方，还导出了`lds文件`中定义的`__setup_start`和`__setup_end`变量。
```c
//In include/linux/init.h
struct obs_kernel_param {
	const char *str;
	int (*setup_func)(char *);
	int early;
};

extern const struct obs_kernel_param __setup_start[], __setup_end[];
```

通过搜索找到在`drivers/tty/serial/earlycon.c`文件中有调用到`early_param`这个宏定义：
```c
/* early_param wrapper for setup_earlycon() */
static int __init param_setup_earlycon(char *buf)
{
	int err;

	/* Just 'earlycon' is a valid param for devicetree and ACPI SPCR. */
	if (!buf || !buf[0]) {
		if (IS_ENABLED(CONFIG_ACPI_SPCR_TABLE)) {
			earlycon_acpi_spcr_enable = true;
			return 0;
		} else if (!buf) {
			return early_init_dt_scan_chosen_stdout();
		}
	}

	err = setup_earlycon(buf);
	if (err == -ENOENT || err == -EALREADY)
		return 0;
	return err;
}
early_param("earlycon", param_setup_earlycon);
```

这里就是将`param_setup_early`这个函数作为结构体的`setup_func`成员的值，并且将这个结构体加入`.init.setup`段，也就是在`__setup_start`和`__setup_end`地址中间，其中涉及到的名为`param_setup_earlycon`的函数，这个留作后话。

### do_early_param

经过一系列的追查，我们终于可以开始分析`do_early_param`，再回顾一下代码：

```c
// In arch/riscv/kernel/setup.c
static int __init do_early_param(char *param, char *val,
				 const char *unused, void *arg)
{
	const struct obs_kernel_param *p;

	for (p = __setup_start; p < __setup_end; p++) {
		if ((p->early && parameq(param, p->str)) ||
		    (strcmp(param, "console") == 0 &&
		     strcmp(p->str, "earlycon") == 0)
		) {
			if (p->setup_func(val) != 0)
				pr_warn("Malformed early option '%s'\n", param);
		}
	}
	/* We accept everything at this stage. */
	return 0;
}
```
可以发现这里定义了一个名为p的指针，类型就是刚刚我们看的`obs_kernel_param`，同时`earlycon.c`中声明的结构体也是这个类型，由于`early_param`或者`__setup`宏能够将结构体加入`__setup_start`和`__setup_end`中间的`.init.setup段`中，所以我们只需要将`p指针`指向`__setup_start`，然后开始遍历，即可获取到在该段中的每一个结构体。

这里判断`early`是否为`1`，也就是只有通过`early_param宏`声明的结构体才可以在这里被展开继续执行，`__setup`的并不可以。

`parameq`函数是判断两个字符串是否相等，并且把`-`替换成`_`去比较的，具体代码如下：

```c
static char dash2underscore(char c)
{
	if (c == '-')
		return '_';
	return c;
}

bool parameqn(const char *a, const char *b, size_t n)
{
	size_t i;

	for (i = 0; i < n; i++) {
		if (dash2underscore(a[i]) != dash2underscore(b[i]))
			return false;
	}
	return true;
}

bool parameq(const char *a, const char *b)
{
	return parameqn(a, b, strlen(a)+1);
}
```
如果判断字符串相等或者`param`为`console`并且`p->str`为`earlycon`就可以执行`setup_func`函数，参数是`val`。

### param_setup_earlycon

经过了上面的执行，我们应该执行到了`setup`函数，也就是`param_setup_earlycon`函数，函数实体如下：

```c
/* early_param wrapper for setup_earlycon() */
static int __init param_setup_earlycon(char *buf)
{
	int err;

	/* Just 'earlycon' is a valid param for devicetree and ACPI SPCR. */
	if (!buf || !buf[0]) {
		if (IS_ENABLED(CONFIG_ACPI_SPCR_TABLE)) {
			earlycon_acpi_spcr_enable = true;
			return 0;
		} else if (!buf) {
			return early_init_dt_scan_chosen_stdout();
		}
	}

	err = setup_earlycon(buf);
	if (err == -ENOENT || err == -EALREADY)
		return 0;
	return err;
}
```

不难看出这就是一个包装函数，用来包装两种处理方式，一种是通过`boot command line`来获取`earlycon`的配置，一种是通过`device tree`来获取配置。

在if判断中判断是否buf为野指针或者buf为空，如果是的话判断`CONFIG_ACPI_SPCR_TABLE`这个宏定义是否开启，如果没开启的话就去搜索`设备树`。否则通过`setup_earlycon`来进行初始化。

## 通过setup_earlycon初始化

`setup_earlycon`函数实体如下：

```c
/**
 *	setup_earlycon - match and register earlycon console
 *	@buf:	earlycon param string
 *
 *	Registers the earlycon console matching the earlycon specified
 *	in the param string @buf. Acceptable param strings are of the form
 *	   <name>,io|mmio|mmio32|mmio32be,<addr>,<options>
 *	   <name>,0x<addr>,<options>
 *	   <name>,<options>
 *	   <name>
 *
 *	Only for the third form does the earlycon setup() method receive the
 *	<options> string in the 'options' parameter; all other forms set
 *	the parameter to NULL.
 *
 *	Returns 0 if an attempt to register the earlycon was made,
 *	otherwise negative error code
 */
int __init setup_earlycon(char *buf)
{
	const struct earlycon_id *match;
	bool empty_compatible = true;

	if (!buf || !buf[0])
		return -EINVAL;

	if (console_is_registered(&early_con))
		return -EALREADY;

again:
	for (match = __earlycon_table; match < __earlycon_table_end; match++) {
		size_t len = strlen(match->name);

		if (strncmp(buf, match->name, len))
			continue;

		/* prefer entries with empty compatible */
		if (empty_compatible && *match->compatible)
			continue;

		if (buf[len]) {
			if (buf[len] != ',')
				continue;
			buf += len + 1;
		} else
			buf = NULL;

		return register_earlycon(buf, match);
	}

	if (empty_compatible) {
		empty_compatible = false;
		goto again;
	}

	return -ENOENT;
}
```
这里对`buf`进行了检验并且检查了`early_con`的`console`是否已经注册过了。
随后使用match变量进行遍历，这其中以`__earlycon_table`地址开始，以`__earlycon_table_end`地址结束。

`__earlycon_table`和`__earlycon_table_end`变量在`arch/riscv/kernel/vmlinux.lds`中可以查找到相关定义，他们是`__earlycon_table`段的开始和结束地址：

```
 .init.data : AT(ADDR(.init.data) - ((((-1))) - 0x80000000 + 1)) 
{  
    __earlycon_table = .; KEEP(*(__earlycon_table)) __earlycon_table_end = .;
}
```

经过搜索发现这两个宏可将内容定义到这个段当中，具体代码如下：

```c
#define OF_EARLYCON_DECLARE(_name, compat, fn)				\
	static const struct earlycon_id __UNIQUE_ID(__earlycon_##_name) \
		EARLYCON_USED_OR_UNUSED  __section("__earlycon_table")  \
		__aligned(__alignof__(struct earlycon_id))		\
		= { .name = __stringify(_name),				\
		    .compatible = compat,				\
		    .setup = fn }

#define EARLYCON_DECLARE(_name, fn)	OF_EARLYCON_DECLARE(_name, "", fn)
```

第一个宏定义`OF_EARLYCON_DECLARE`是用于设备树的
    - `_name`: 名字，不应该有双引号
    - `compat`: 与设备树的compatible相对应，用于匹配，应有双引号
    - `fn`: 执行函数，即匹配到后执行的函数

第二个宏定义`EARLYCON_DECLARE`是用于非设备树的版本的，参数定义与`OF_EARLYCON_DECLARE`相同，只不过将compat设置为了空字符串也就是`\0`。

拿`bpi-f3`为例，其在`drivers/tty/serial/pxa_k1x.c`中有如下定义：

```c
/* Support for earlycon */
static void pxa_early_write(struct console *con, const char *s,
			unsigned n)
{
	   struct earlycon_device *dev = con->data;

	   uart_console_write(&dev->port, s, n, serial_pxa_console_putchar);
}

static int __init pxa_early_console_setup(struct earlycon_device *device, const char *opt)
{
	if (!device->port.membase) {
		return -ENODEV;
	}

	device->con->write = pxa_early_write;
	return 0;
}

EARLYCON_DECLARE(pxa_serial, pxa_early_console_setup);
```
这部分代码将`pxa_serial`, `""`, `pxa_early_console_setup`依次定义成结构体。

`earlycon_id`的成员并且放在__earlycon_table段当中，`earlycon_id`结构体定义如下：

```c
struct earlycon_id {
	char	name[15];
	char	name_term;	/* In case compiler didn't '\0' term name */
	char	compatible[128];
	int	(*setup)(struct earlycon_device *, const char *options);
};
```

其中, `name_term`成员是为了`name`字符串以`\0`为结尾。

继续回到`setup_earlycon`函数：

```c
/**
 *	setup_earlycon - match and register earlycon console
 *	@buf:	earlycon param string
 *
 *	Registers the earlycon console matching the earlycon specified
 *	in the param string @buf. Acceptable param strings are of the form
 *	   <name>,io|mmio|mmio32|mmio32be,<addr>,<options>
 *	   <name>,0x<addr>,<options>
 *	   <name>,<options>
 *	   <name>
 *
 *	Only for the third form does the earlycon setup() method receive the
 *	<options> string in the 'options' parameter; all other forms set
 *	the parameter to NULL.
 *
 *	Returns 0 if an attempt to register the earlycon was made,
 *	otherwise negative error code
 */
int __init setup_earlycon(char *buf)
{
	const struct earlycon_id *match;
	bool empty_compatible = true;

	if (!buf || !buf[0])
		return -EINVAL;

	if (console_is_registered(&early_con))
		return -EALREADY;

again:
	for (match = __earlycon_table; match < __earlycon_table_end; match++) {
		size_t len = strlen(match->name);

		if (strncmp(buf, match->name, len))
			continue;

		/* prefer entries with empty compatible */
		if (empty_compatible && *match->compatible)
			continue;

		if (buf[len]) {
			if (buf[len] != ',')
				continue;
			buf += len + 1;
		} else
			buf = NULL;

		return register_earlycon(buf, match);
	}

	if (empty_compatible) {
		empty_compatible = false;
		goto again;
	}

	return -ENOENT;
}
```

这里比较`buf`与每一个在`__earlycon_table`段中的`earlycon_id`结构体的`name`是否匹配。

在匹配之后比较每一个在`__earlycon_table`段中的`earlycon_id`结构体的`compatible`是否不为`空`。

随后进行判断是否有`,`存在，如果有`,`存在，就会跳过前面的内容，否则则设置为`NULL`。

e.g. :
```
buf="pxa_serial,0xd4017000"
    ->
        buf="0xd4017000"
```

最后通过`register_earlycon`函数进行注册：

```c
static int __init register_earlycon(char *buf, const struct earlycon_id *match)
{
	int err;
	struct uart_port *port = &early_console_dev.port;

	/* On parsing error, pass the options buf to the setup function */
	if (buf && !parse_options(&early_console_dev, buf))
		buf = NULL;

	spin_lock_init(&port->lock);
	if (!port->uartclk)
		port->uartclk = BASE_BAUD * 16;
	if (port->mapbase)
		port->membase = earlycon_map(port->mapbase, 64);

	earlycon_init(&early_console_dev, match->name);
	err = match->setup(&early_console_dev, buf);
	earlycon_print_info(&early_console_dev);
	if (err < 0)
		return err;
	if (!early_console_dev.con->write)
		return -ENODEV;

	register_console(early_console_dev.con);
	return 0;
}
```

通过`parse_options`解析参数值：

```c
//In drivers/tty/serial/earlycon.c
static int __init parse_options(struct earlycon_device *device, char *options)
{
	struct uart_port *port = &device->port;
	int length;
	resource_size_t addr;

	if (uart_parse_earlycon(options, &port->iotype, &addr, &options))
		return -EINVAL;

	switch (port->iotype) {
	case UPIO_MEM:
		port->mapbase = addr;
		break;
	case UPIO_MEM16:
		port->regshift = 1;
		port->mapbase = addr;
		break;
	case UPIO_MEM32:
	case UPIO_MEM32BE:
		port->regshift = 2;
		port->mapbase = addr;
		break;
	case UPIO_PORT:
		port->iobase = addr;
		break;
	default:
		return -EINVAL;
	}

	if (options) {
		char *uartclk;

		device->baud = simple_strtoul(options, NULL, 0);
		uartclk = strchr(options, ',');
		if (uartclk && kstrtouint(uartclk + 1, 0, &port->uartclk) < 0)
			pr_warn("[%s] unsupported earlycon uart clkrate option\n",
				options);
		length = min(strcspn(options, " ") + 1,
			     (size_t)(sizeof(device->options)));
		strscpy(device->options, options, length);
	}

	return 0;
}
```

通过`uart_parse_earlycon`函数解析传入参数`options`，也就是`buf`，`buf`指针现在指向`0xd4017000`。

```c
// In drivers/tty/serial/serial_core.c

/**
 * uart_parse_earlycon - Parse earlycon options
 * @p:	     ptr to 2nd field (ie., just beyond '<name>,')
 * @iotype:  ptr for decoded iotype (out)
 * @addr:    ptr for decoded mapbase/iobase (out)
 * @options: ptr for <options> field; %NULL if not present (out)
 *
 * Decodes earlycon kernel command line parameters of the form:
 *  * earlycon=<name>,io|mmio|mmio16|mmio32|mmio32be|mmio32native,<addr>,<options>
 *  * console=<name>,io|mmio|mmio16|mmio32|mmio32be|mmio32native,<addr>,<options>
 *
 * The optional form:
 *  * earlycon=<name>,0x<addr>,<options>
 *  * console=<name>,0x<addr>,<options>
 *
 * is also accepted; the returned @iotype will be %UPIO_MEM.
 *
 * Returns: 0 on success or -%EINVAL on failure
 */
int uart_parse_earlycon(char *p, unsigned char *iotype, resource_size_t *addr,
			char **options)
{
	if (strncmp(p, "mmio,", 5) == 0) {
		*iotype = UPIO_MEM;
		p += 5;
	} else if (strncmp(p, "mmio16,", 7) == 0) {
		*iotype = UPIO_MEM16;
		p += 7;
	} else if (strncmp(p, "mmio32,", 7) == 0) {
		*iotype = UPIO_MEM32;
		p += 7;
	} else if (strncmp(p, "mmio32be,", 9) == 0) {
		*iotype = UPIO_MEM32BE;
		p += 9;
	} else if (strncmp(p, "mmio32native,", 13) == 0) {
		*iotype = IS_ENABLED(CONFIG_CPU_BIG_ENDIAN) ?
			UPIO_MEM32BE : UPIO_MEM32;
		p += 13;
	} else if (strncmp(p, "io,", 3) == 0) {
		*iotype = UPIO_PORT;
		p += 3;
	} else if (strncmp(p, "0x", 2) == 0) {
		*iotype = UPIO_MEM;
	} else {
		return -EINVAL;
	}

	/*
	 * Before you replace it with kstrtoull(), think about options separator
	 * (',') it will not tolerate
	 */
	*addr = simple_strtoull(p, NULL, 0);
	p = strchr(p, ',');
	if (p)
		p++;

	*options = p;
	return 0;
}
```

不难看出这里在对`io类型`进行判断，我们`buf`指向的内容是`0xd4017000`，所以这里`iotype`被设置为了`UPIO_MEM`。

通过`simple_strtoull`函数将字符串转为数字，此时`addr`应该为`0xd4017000`。

`strchr`函数找出是否有额外选项，如果有额外选项，则将额外选项开始的字符串地址赋值给`options`指针，这里我们并没有，所以应该指向了`NULL`。

回到`parse_options`函数：

```c
//In drivers/tty/serial/earlycon.c
static int __init parse_options(struct earlycon_device *device, char *options)
{
	struct uart_port *port = &device->port;
	int length;
	resource_size_t addr;

	if (uart_parse_earlycon(options, &port->iotype, &addr, &options))
		return -EINVAL;

	switch (port->iotype) {
	case UPIO_MEM:
		port->mapbase = addr;
		break;
	case UPIO_MEM16:
		port->regshift = 1;
		port->mapbase = addr;
		break;
	case UPIO_MEM32:
	case UPIO_MEM32BE:
		port->regshift = 2;
		port->mapbase = addr;
		break;
	case UPIO_PORT:
		port->iobase = addr;
		break;
	default:
		return -EINVAL;
	}

	if (options) {
		char *uartclk;

		device->baud = simple_strtoul(options, NULL, 0);
		uartclk = strchr(options, ',');
		if (uartclk && kstrtouint(uartclk + 1, 0, &port->uartclk) < 0)
			pr_warn("[%s] unsupported earlycon uart clkrate option\n",
				options);
		length = min(strcspn(options, " ") + 1,
			     (size_t)(sizeof(device->options)));
		strscpy(device->options, options, length);
	}

	return 0;
}
```

经过以上的步骤，`port->iotype`的类型应该被设置成了`UPIO_MEM`，所以将`port->mapbase`设置成了`addr`。

`if`判断`options`指针是否为空，也就是是否有额外选项。

回到`register_earlycon`函数：

```c
static int __init register_earlycon(char *buf, const struct earlycon_id *match)
{
	int err;
	struct uart_port *port = &early_console_dev.port;

	/* On parsing error, pass the options buf to the setup function */
	if (buf && !parse_options(&early_console_dev, buf))
		buf = NULL;

	spin_lock_init(&port->lock);
	if (!port->uartclk)
		port->uartclk = BASE_BAUD * 16;
	if (port->mapbase)
		port->membase = earlycon_map(port->mapbase, 64);

	earlycon_init(&early_console_dev, match->name);
	err = match->setup(&early_console_dev, buf);
	earlycon_print_info(&early_console_dev);
	if (err < 0)
		return err;
	if (!early_console_dev.con->write)
		return -ENODEV;

	register_console(early_console_dev.con);
	return 0;
}
```

通过`parse_options`函数执行后，`mapbase`, `iotype`都已经被设置好，但由于我们没有`options`选项，所以`uartclk`是没有被设置的，可以看到如果这里没有被设置的话，在这里就会被手动设置为`BASE_BAUD * 16`。

`mapbase`也就是串口寄存器地址的地址我们现在已经拿到了，但是这是个物理地址，所以要通过`earlycon_map`函数映射进入页表，随后返回虚拟地址赋值给`membase`。

调用`earlycon_init`函数进行赋值，随后调用`match->setup`函数，也就是通过`EARLYCON_DECALRE`宏声明的函数`pxa_early_console_setup`：

```c
static int __init pxa_early_console_setup(struct earlycon_device *device, const char *opt)
{
	if (!device->port.membase) {
		return -ENODEV;
	}

	device->con->write = pxa_early_write;
	return 0;
}

EARLYCON_DECLARE(pxa_serial, pxa_early_console_setup);
```

可以看到这个函数就是判断`membase`也就是虚拟地址是否映射成功，并且设置了`write`函数为`pxa_early_write`

```c
/* Support for earlycon */
static void pxa_early_write(struct console *con, const char *s,
			unsigned n)
{
	   struct earlycon_device *dev = con->data;

	   uart_console_write(&dev->port, s, n, serial_pxa_console_putchar);
}
```

## 通过early_init_dt_scan_chosen_stdout初始化
#### TODO


```c
// In drivers/of/fdt.c

#ifdef CONFIG_SERIAL_EARLYCON

int __init early_init_dt_scan_chosen_stdout(void)
{
	int offset;
	const char *p, *q, *options = NULL;
	int l;
	const struct earlycon_id *match;
	const void *fdt = initial_boot_params;
	int ret;

	offset = fdt_path_offset(fdt, "/chosen");
	if (offset < 0)
		offset = fdt_path_offset(fdt, "/chosen@0");
	if (offset < 0)
		return -ENOENT;

	p = fdt_getprop(fdt, offset, "stdout-path", &l);
	if (!p)
		p = fdt_getprop(fdt, offset, "linux,stdout-path", &l);
	if (!p || !l)
		return -ENOENT;

	q = strchrnul(p, ':');
	if (*q != '\0')
		options = q + 1;
	l = q - p;

	/* Get the node specified by stdout-path */
	offset = fdt_path_offset_namelen(fdt, p, l);
	if (offset < 0) {
		pr_warn("earlycon: stdout-path %.*s not found\n", l, p);
		return 0;
	}

	for (match = __earlycon_table; match < __earlycon_table_end; match++) {
		if (!match->compatible[0])
			continue;

		if (fdt_node_check_compatible(fdt, offset, match->compatible))
			continue;

		ret = of_setup_earlycon(match, offset, options);
		if (!ret || ret == -EALREADY)
			return 0;
	}
	return -ENODEV;
}
#endif
```
