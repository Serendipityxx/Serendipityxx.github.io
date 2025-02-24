---
layout:  post
title:   "在postgreSQL中使用guc参数"
date:   2024-10-30 13:48:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   false
tags:
- c
- guc参数
- postgres内核
---

# 1 guc参数
guc模块实现了多种参数类型，目前包括以下几种：
```c
const char *const config_type_names[] =
{
	 /* PGC_BOOL */ "bool",
	 /* PGC_INT */ "integer",
	 /* PGC_REAL */ "real",
	 /* PGC_STRING */ "string",
	 /* PGC_ENUM */ "enum"
};
```
bool类型、整数类型、实数类型、字符串类型和枚举类型五种。

在定义完这些参数类型之后，还有一个重要的标识就是这些参数在什么时候才能生效，如果大家做过dba的话对这些比较了解，参数生效的时机也是需要进行控制的，比如说有些参数设置完之后可以立即生效，但是有的参数设置完之后需要重启数据库服务等等，这些都是有所差别的，具体的生效的类型有以下几种：
```c
typedef enum
{
	PGC_INTERNAL,           //在内核中设置，如server_version
	PGC_POSTMASTER,         //在postmaster启动的时候设置，配置之后需要重启数据库服务
	PGC_SIGHUP,             //只能在postmaster启动时设置，或者通过更改配置文件并向postmaster或后台进程发送HUP信号来设置。
	PGC_SU_BACKEND,         //SU_BACKEND选项只能在用户是超级用户，postmaster设置或者sighup方式进行设置
	PGC_BACKEND,            //postmaster启动时或者发送sighup通知
	PGC_SUSET,              //postmaster启动的时候或者超级用户通过sql设置
	PGC_USERSET             //任何用户在任何时候都可以设置
} GucContext;
```
postmaster配置guc参数的基本过程包括：

- 初始化guc参数，将参数设置为默认值
- 配置guc参数，根据命令行参数配置参数
- 读取配置文件重新设置参数

在使用的过程中最后一步就是设置这个参数，pg根据所选的参数类型不同，在设置方面也有所不同，如下设置：
```c
struct config_bool
{
	struct config_generic gen;
	/* constant fields, must be set correctly in initial value: */
	bool	   *variable;
	bool		boot_val;
	GucBoolCheckHook check_hook;
	GucBoolAssignHook assign_hook;
	GucShowHook show_hook;
	/* variable fields, initialized at runtime: */
	bool		reset_val;
	void	   *reset_extra;
};

struct config_int
{
	struct config_generic gen;
	/* constant fields, must be set correctly in initial value: */
	int		   *variable;
	int			boot_val;
	int			min;
	int			max;
	GucIntCheckHook check_hook;
	GucIntAssignHook assign_hook;
	GucShowHook show_hook;
	/* variable fields, initialized at runtime: */
	int			reset_val;
	void	   *reset_extra;
};

struct config_real
{
	struct config_generic gen;
	/* constant fields, must be set correctly in initial value: */
	double	   *variable;
	double		boot_val;
	double		min;
	double		max;
	GucRealCheckHook check_hook;
	GucRealAssignHook assign_hook;
	GucShowHook show_hook;
	/* variable fields, initialized at runtime: */
	double		reset_val;
	void	   *reset_extra;
};

/*
 * A note about string GUCs: the boot_val is allowed to be NULL, which leads
 * to the reset_val and the actual variable value (*variable) also being NULL.
 * However, there is no way to set a NULL value subsequently using
 * set_config_option or any other GUC API.  Also, GUC APIs such as SHOW will
 * display a NULL value as an empty string.  Callers that choose to use a NULL
 * boot_val should overwrite the setting later in startup, or else be careful
 * that NULL doesn't have semantics that are visibly different from an empty
 * string.
 */
struct config_string
{
	struct config_generic gen;
	/* constant fields, must be set correctly in initial value: */
	char	  **variable;
	const char *boot_val;
	GucStringCheckHook check_hook;
	GucStringAssignHook assign_hook;
	GucShowHook show_hook;
	/* variable fields, initialized at runtime: */
	char	   *reset_val;
	void	   *reset_extra;
};

struct config_enum
{
	struct config_generic gen;
	/* constant fields, must be set correctly in initial value: */
	int		   *variable;
	int			boot_val;
	const struct config_enum_entry *options;
	GucEnumCheckHook check_hook;
	GucEnumAssignHook assign_hook;
	GucShowHook show_hook;
	/* variable fields, initialized at runtime: */
	int			reset_val;
	void	   *reset_extra;
};
```
这里的`config_generic`类型是适用于所有类型的通用字段，具体成员解释如下：
```c
struct config_generic
{
	/* constant fields, must be set correctly in initial value: */
	const char *name;			/* name of variable - MUST BE FIRST 变量的名字 */
	GucContext	context;		/* context required to set the variable 设置变量要求的上下文 */
	enum config_group group;	/* to help organize variables by function 根据功能对参数分组 */
	const char *short_desc;		/* short desc. of this variable's purpose 简短描述 */
	const char *long_desc;		/* long desc. of this variable's purpose 详细描述 */
	int			flags;			/* flag bits, see guc.h  参数标志 */
	/* variable fields, initialized at runtime: */
	enum config_type vartype;	/* type of variable (set only at startup) 参数值的数据类型 */
	int			status;			/* status bits, see below 参数的状态 */
	GucSource	source;			/* source of the current actual value 当前参数来源 */
	GucSource	reset_source;	/* source of the reset_value 参数值为reset_value的时的来源 */
	GucContext	scontext;		/* context that set the current value 当前参数操作源 */
	GucContext	reset_scontext; /* context that set the reset value 重置参数操作源 */
	Oid			srole;			/* role that set the current value 设置当前值的角色oid */
	Oid			reset_srole;	/* role that set the reset value 重置值的角色oid */
	GucStack   *stack;			/* stacked prior values 保存旧值支持回滚 */
	void	   *extra;			/* "extra" pointer for current actual value 当前值的extra指针 */
	char	   *last_reported;	/* if variable is GUC_REPORT, value last sent 如果变量是GUC_REPORT，上次发送给客户端的值（如果还没有发送，则为NULL）
								 * to client (NULL if not yet sent) */
	char	   *sourcefile;		/* file current setting is from (NULL if not
								 * set in config file) 配置所在的源文件 */
	int			sourceline;		/* line in source file 源文件中的行号 */
};
```
在上面的GucSource参数来源枚举类型字段解释：
```c
typedef enum {

    PGC_S_DEFAULT, //缺省配置

    PGC_S_DYNAMIC_DEFAULT, //初始化时动态计算

    PGC_S_ENV_VAR, //环境变量

    PGC_S_FILE, // postgresql.conf

    PGC_S_ARGV, //命令行参数

    PGC_S_GLOBAL, //数据库全局设置

    PGC_S_DATABASE, //数据库安装时指定

    PGC_S_USER, //用户指定

    PGC_S_DATABASE_USER, //数据库安装时用户指定

    PGC_S_CLIENT,//客户端连接请求

    PGC_S_OVERRIDE, //特殊情况强制设置为默认值

    PGC_S_INTERACTIVE, //错误报告的分割符

    PGC_S_TEST, //仅用于测试

    PGC_S_SESSION //set命令

} GucSource;
```
可以参考下面的eg：

![alt text](/img/postimg/image1.jpg)

在使用之前首先需要先定义一个全局的参数类型！
## 1.1 初始化guc参数
postmaster调用`InitializeGUCOptions`函数将参数设置为默认值:
```c
void
InitializeGUCOptions(void)
{
	int			i;

	/*
	 * Before log_line_prefix could possibly receive a nonempty setting, make
	 * sure that timezone processing is minimally alive (see elog.c).
	 */
	pg_timezone_initialize();

	/*
	 * Build sorted array of all GUC variables.
	 */
	build_guc_variables();

	/*
	 * Load all variables with their compiled-in defaults, and initialize
	 * status fields as needed.
	 */
	for (i = 0; i < num_guc_variables; i++)
	{
		InitializeOneGUCOption(guc_variables[i]);
	}

	guc_dirty = false;

	reporting_enabled = false;

	/*
	 * Prevent any attempt to override the transaction modes from
	 * non-interactive sources.
	 */
	SetConfigOption("transaction_isolation", "read committed",
					PGC_POSTMASTER, PGC_S_OVERRIDE);
	SetConfigOption("transaction_read_only", "no",
					PGC_POSTMASTER, PGC_S_OVERRIDE);
	SetConfigOption("transaction_deferrable", "no",
					PGC_POSTMASTER, PGC_S_OVERRIDE);

	/*
	 * For historical reasons, some GUC parameters can receive defaults from
	 * environment variables.  Process those settings.
	 */
	InitializeGUCOptionsFromEnvironment();
}
```
首先调用build_guc_variables函数来统计参数个数并分配相应的config_generic类型的全局指针数组guc_variables以保存每个参数结构体的地址，并且对该数据进行排序。由于参数是通过全局静态数组ConfigureNamesBool、ConfigureNamesInt、ConfigureNamesReal、ConfigureNamesString、ConfigureNamesEnum存储的，因此在build_guc_variables函数中只需要遍历相应的数组，统计参数的个数并将参数结构体中config_generic域的参数vartyoe设置为相应的参数类型。当遍历完所有参数后，根据总的参数个数分配config_generic指针数组guc_vars，然后再次遍历静态参数数组，将每个参数结构的首地址保存到guc_vars数组中（这里分配的数组个数为当前参数总数的1.25倍，主要是为了方便以后参数的扩充）。接着将全局变量guc_variables也指向guc_vars数组。最后通过快速排序法把guc_variables按照参数名进行排序。
```c
void
build_guc_variables(void)
{
	int			size_vars;
	int			num_vars = 0;
	struct config_generic **guc_vars;
	int			i;

	for (i = 0; ConfigureNamesBool[i].gen.name; i++)
	{
		struct config_bool *conf = &ConfigureNamesBool[i];

		/* Rather than requiring vartype to be filled in by hand, do this: */
		conf->gen.vartype = PGC_BOOL;
		num_vars++;
	}

	for (i = 0; ConfigureNamesInt[i].gen.name; i++)
	{
		struct config_int *conf = &ConfigureNamesInt[i];

		conf->gen.vartype = PGC_INT;
		num_vars++;
	}

	for (i = 0; ConfigureNamesReal[i].gen.name; i++)
	{
		struct config_real *conf = &ConfigureNamesReal[i];

		conf->gen.vartype = PGC_REAL;
		num_vars++;
	}

	for (i = 0; ConfigureNamesString[i].gen.name; i++)
	{
		struct config_string *conf = &ConfigureNamesString[i];

		conf->gen.vartype = PGC_STRING;
		num_vars++;
	}

	for (i = 0; ConfigureNamesEnum[i].gen.name; i++)
	{
		struct config_enum *conf = &ConfigureNamesEnum[i];

		conf->gen.vartype = PGC_ENUM;
		num_vars++;
	}

	/*
	 * Create table with 20% slack
	 */
	size_vars = num_vars + num_vars / 4;

	guc_vars = (struct config_generic **)
		guc_malloc(FATAL, size_vars * sizeof(struct config_generic *));

	num_vars = 0;

	for (i = 0; ConfigureNamesBool[i].gen.name; i++)
		guc_vars[num_vars++] = &ConfigureNamesBool[i].gen;

	for (i = 0; ConfigureNamesInt[i].gen.name; i++)
		guc_vars[num_vars++] = &ConfigureNamesInt[i].gen;

	for (i = 0; ConfigureNamesReal[i].gen.name; i++)
		guc_vars[num_vars++] = &ConfigureNamesReal[i].gen;

	for (i = 0; ConfigureNamesString[i].gen.name; i++)
		guc_vars[num_vars++] = &ConfigureNamesString[i].gen;

	for (i = 0; ConfigureNamesEnum[i].gen.name; i++)
		guc_vars[num_vars++] = &ConfigureNamesEnum[i].gen;

	if (guc_variables)
		free(guc_variables);
	guc_variables = guc_vars;
	num_guc_variables = num_vars;
	size_guc_variables = size_vars;
	qsort((void *) guc_variables, num_guc_variables,
		  sizeof(struct config_generic *), guc_var_compare);
}
```
再根据参数的类型不同，对这种类型的guc参数的其他属性值进行设置，将`reset_source、tentative_source、source`设置为`PGC_S_DEFAULT`表示默认；`stack、sourcefile`设置为`NULL`；然后根据参数值vartype的不同类型分别调用相应的`assign_hook`函数（如果该参数设置了该函数），`assign_hook`函数用来设置boot_val，最后将boot_val赋值给reset_val和variable指向的变量，通过这样一系列的步骤就将参数设置为了默认值。
```c
static void
InitializeOneGUCOption(struct config_generic *gconf)
{
	gconf->status = 0;
	gconf->source = PGC_S_DEFAULT;
	gconf->reset_source = PGC_S_DEFAULT;
	gconf->scontext = PGC_INTERNAL;
	gconf->reset_scontext = PGC_INTERNAL;
	gconf->srole = BOOTSTRAP_SUPERUSERID;
	gconf->reset_srole = BOOTSTRAP_SUPERUSERID;
	gconf->stack = NULL;
	gconf->extra = NULL;
	gconf->last_reported = NULL;
	gconf->sourcefile = NULL;
	gconf->sourceline = 0;

	switch (gconf->vartype)
	{
		case PGC_BOOL:
			{
				struct config_bool *conf = (struct config_bool *) gconf;
				bool		newval = conf->boot_val;
				void	   *extra = NULL;

				if (!call_bool_check_hook(conf, &newval, &extra,
										  PGC_S_DEFAULT, LOG))
					elog(FATAL, "failed to initialize %s to %d",
						 conf->gen.name, (int) newval);
				if (conf->assign_hook)
					conf->assign_hook(newval, extra);
				*conf->variable = conf->reset_val = newval;
				conf->gen.extra = conf->reset_extra = extra;
				break;
			}
		case PGC_INT:
			{
				struct config_int *conf = (struct config_int *) gconf;
				int			newval = conf->boot_val;
				void	   *extra = NULL;

				Assert(newval >= conf->min);
				Assert(newval <= conf->max);
				if (!call_int_check_hook(conf, &newval, &extra,
										 PGC_S_DEFAULT, LOG))
					elog(FATAL, "failed to initialize %s to %d",
						 conf->gen.name, newval);
				if (conf->assign_hook)
					conf->assign_hook(newval, extra);
				*conf->variable = conf->reset_val = newval;
				conf->gen.extra = conf->reset_extra = extra;
				break;
			}
		case PGC_REAL:
			{
				struct config_real *conf = (struct config_real *) gconf;
				double		newval = conf->boot_val;
				void	   *extra = NULL;

				Assert(newval >= conf->min);
				Assert(newval <= conf->max);
				if (!call_real_check_hook(conf, &newval, &extra,
										  PGC_S_DEFAULT, LOG))
					elog(FATAL, "failed to initialize %s to %g",
						 conf->gen.name, newval);
				if (conf->assign_hook)
					conf->assign_hook(newval, extra);
				*conf->variable = conf->reset_val = newval;
				conf->gen.extra = conf->reset_extra = extra;
				break;
			}
		case PGC_STRING:
			{
				struct config_string *conf = (struct config_string *) gconf;
				char	   *newval;
				void	   *extra = NULL;

				/* non-NULL boot_val must always get strdup'd */
				if (conf->boot_val != NULL)
					newval = guc_strdup(FATAL, conf->boot_val);
				else
					newval = NULL;

				if (!call_string_check_hook(conf, &newval, &extra,
											PGC_S_DEFAULT, LOG))
					elog(FATAL, "failed to initialize %s to \"%s\"",
						 conf->gen.name, newval ? newval : "");
				if (conf->assign_hook)
					conf->assign_hook(newval, extra);
				*conf->variable = conf->reset_val = newval;
				conf->gen.extra = conf->reset_extra = extra;
				break;
			}
		case PGC_ENUM:
			{
				struct config_enum *conf = (struct config_enum *) gconf;
				int			newval = conf->boot_val;
				void	   *extra = NULL;

				if (!call_enum_check_hook(conf, &newval, &extra,
										  PGC_S_DEFAULT, LOG))
					elog(FATAL, "failed to initialize %s to %d",
						 conf->gen.name, newval);
				if (conf->assign_hook)
					conf->assign_hook(newval, extra);
				*conf->variable = conf->reset_val = newval;
				conf->gen.extra = conf->reset_extra = extra;
				break;
			}
	}
}
```
然后就是执行`InitializeGUCOptionsFromEnvironment`函数，首先是从环境变量中获取这三个参数的值，如果不为空的话就使用`SetConfigOption`函数设置这三个环境变量对应的guc参数的值。

然后再检查系统的最大安全栈深度，如果这个值在100KB和2MB之间，则使用其作为`max_stack_depth`的值。

```c
static void
InitializeGUCOptionsFromEnvironment(void)
{
	char	   *env;
	long		stack_rlimit;

	env = getenv("PGPORT");
	if (env != NULL)
		SetConfigOption("port", env, PGC_POSTMASTER, PGC_S_ENV_VAR);

	env = getenv("PGDATESTYLE");
	if (env != NULL)
		SetConfigOption("datestyle", env, PGC_POSTMASTER, PGC_S_ENV_VAR);

	env = getenv("PGCLIENTENCODING");
	if (env != NULL)
		SetConfigOption("client_encoding", env, PGC_POSTMASTER, PGC_S_ENV_VAR);

	/*
	 * rlimit isn't exactly an "environment variable", but it behaves about
	 * the same.  If we can identify the platform stack depth rlimit, increase
	 * default stack depth setting up to whatever is safe (but at most 2MB).
	 * Report the value's source as PGC_S_DYNAMIC_DEFAULT if it's 2MB, or as
	 * PGC_S_ENV_VAR if it's reflecting the rlimit limit.
	 */
	stack_rlimit = get_stack_depth_rlimit();
	if (stack_rlimit > 0)
	{
		long		new_limit = (stack_rlimit - STACK_DEPTH_SLOP) / 1024L;

		if (new_limit > 100)
		{
			GucSource	source;
			char		limbuf[16];

			if (new_limit < 2048)
				source = PGC_S_ENV_VAR;
			else
			{
				new_limit = 2048;
				source = PGC_S_DYNAMIC_DEFAULT;
			}
			snprintf(limbuf, sizeof(limbuf), "%ld", new_limit);
			SetConfigOption("max_stack_depth", limbuf,
							PGC_POSTMASTER, source);
		}
	}
}
```
## 1.2 配置guc参数
如果用户启动Postmaster进程时通过命令行参数指定了一些GUC的参数值，那么Postmaster需要从命令行参数中将这些GUC参数的值解析出来并且设置到相应的GUC参数中，这一部分代码在Postmaster.c文件中。根据命令行设置参数主要是通过getopt和SetConfigOption这两个函数来完成的。对于getopt返回的每一个参数选项及其参数，通过一个switch语句根据参数选项的不同分别调用SetConfigOption函数设置相应的参数。
```c
while ((opt = getopt(argc, argv, "B:bc:C:D:d:EeFf:h:ijk:lN:nOPp:r:S:sTt:W:-:")) != -1)
	{
		switch (opt)
		{
			case 'B':
				SetConfigOption("shared_buffers", optarg, PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'b':
				/* Undocumented flag used for binary upgrades */
				IsBinaryUpgrade = true;
				break;

			case 'C':
				output_config_variable = strdup(optarg);
				break;

			case 'D':
				userDoption = strdup(optarg);
				break;

			case 'd':
				set_debug_options(atoi(optarg), PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'E':
				SetConfigOption("log_statement", "all", PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'e':
				SetConfigOption("datestyle", "euro", PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'F':
				SetConfigOption("fsync", "false", PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'f':
				if (!set_plan_disabling_options(optarg, PGC_POSTMASTER, PGC_S_ARGV))
				{
					write_stderr("%s: invalid argument for option -f: \"%s\"\n",
								 progname, optarg);
					ExitPostmaster(1);
				}
				break;

			case 'h':
				SetConfigOption("listen_addresses", optarg, PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'i':
				SetConfigOption("listen_addresses", "*", PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'j':
				/* only used by interactive backend */
				break;

			case 'k':
				SetConfigOption("unix_socket_directories", optarg, PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'l':
				SetConfigOption("ssl", "true", PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'N':
				SetConfigOption("max_connections", optarg, PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'n':
				/* Don't reinit shared mem after abnormal exit */
				Reinit = false;
				break;

			case 'O':
				SetConfigOption("allow_system_table_mods", "true", PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'P':
				SetConfigOption("ignore_system_indexes", "true", PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'p':
				SetConfigOption("port", optarg, PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'r':
				/* only used by single-user backend */
				break;

			case 'S':
				SetConfigOption("work_mem", optarg, PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 's':
				SetConfigOption("log_statement_stats", "true", PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'T':

				/*
				 * In the event that some backend dumps core, send SIGSTOP,
				 * rather than SIGQUIT, to all its peers.  This lets the wily
				 * post_hacker collect core dumps from everyone.
				 */
				SendStop = true;
				break;

			case 't':
				{
					const char *tmp = get_stats_option_name(optarg);

					if (tmp)
					{
						SetConfigOption(tmp, "true", PGC_POSTMASTER, PGC_S_ARGV);
					}
					else
					{
						write_stderr("%s: invalid argument for option -t: \"%s\"\n",
									 progname, optarg);
						ExitPostmaster(1);
					}
					break;
				}

			case 'W':
				SetConfigOption("post_auth_delay", optarg, PGC_POSTMASTER, PGC_S_ARGV);
				break;

			case 'c':
			case '-':
				{
					char	   *name,
							   *value;

					ParseLongOption(optarg, &name, &value);
					if (!value)
					{
						if (opt == '-')
							ereport(ERROR,
									(errcode(ERRCODE_SYNTAX_ERROR),
									 errmsg("--%s requires a value",
											optarg)));
						else
							ereport(ERROR,
									(errcode(ERRCODE_SYNTAX_ERROR),
									 errmsg("-c %s requires a value",
											optarg)));
					}

					SetConfigOption(name, value, PGC_POSTMASTER, PGC_S_ARGV);
					free(name);
					if (value)
						free(value);
					break;
				}

			default:
				write_stderr("Try \"%s --help\" for more information.\n",
							 progname);
				ExitPostmaster(1);
		}
	}
```
SetConfigOption函数的第一个参数为参数名，第二个参数为参数值，其值存放在getopt函数返回的optarg字符串中，第三个参数是参数类型，第四个参数为参数来源。由于在这里Postmaster是在处理命令行参数，所以这里的参数类型和参数来源分别设置为PGC_POSTMASTER何PGC_S_ARGV。SetConfigOption函数是通过set_config_option(const char \*name, const char\* value, GucContext context, GucSource source, bool isLocal, bool changeVal)函数来实现的，其中最后2个参数统一设置为false和true。该函数首先从guc_variables指向的参数数组中搜索参数名为name的参数，如果没有找到则出错；否则将找到的参数的结构体中GucContext的值与传过来的参数context比较，判断在当前的上下文中参数是否可以设置，如果不能设置的话就报错，否则再将参数结构体中的GucSource与传过来的参数source进行比较，判断当前操作的优先级是否大于或等于先前的优先级，如果大于或等于先前优先级的话则根据具体参数值的类型将value转化为相应的数据，然后设置参数结构体中的相应数据项即可。

## 1.3 读取参数配置文件进行设置
当完成命令行参数的设置之后，接着读配置文件重新配置参数。需要注意的是，在配置文件中设置的参数都不能修改之前通过命令行已经设置的参数，因为其优先级没有通过命令行设置的优先级高。这个过程主要是调用SelectConfigFiles(const char* userDoption, const char* progname)函数来实现的，其中第一个参数是通过命令行设置的用户的数据目录，如果没有设置会通过环境变量PG-DATA找到；第二个参数为程序名，主要用于错误处理。该函数首先在数据目录下找到配置文件，然后调用词法分析程序解析文件。对于解析到的每个参数及其参数值，调用SetConfigOption来完成参数的修改。
```c
bool
SelectConfigFiles(const char *userDoption, const char *progname)
{
	char	   *configdir;
	char	   *fname;
	struct stat stat_buf;

	/* configdir is -D option, or $PGDATA if no -D 找到数据目录*/
	if (userDoption)
		configdir = make_absolute_path(userDoption);
	else
		configdir = make_absolute_path(getenv("PGDATA"));

	if (configdir && stat(configdir, &stat_buf) != 0)
	{
		write_stderr("%s: could not access directory \"%s\": %s\n",
					 progname,
					 configdir,
					 strerror(errno));
		if (errno == ENOENT)
			write_stderr("Run initdb or pg_basebackup to initialize a PostgreSQL data directory.\n");
		return false;
	}

	/*
	 * Find the configuration file: if config_file was specified on the
	 * command line, use it, else use configdir/postgresql.conf.  In any case
	 * ensure the result is an absolute path, so that it will be interpreted
	 * the same way by future backends.
	 */
	if (ConfigFileName)
		fname = make_absolute_path(ConfigFileName);
	else if (configdir)
	{
		fname = guc_malloc(FATAL,
						   strlen(configdir) + strlen(CONFIG_FILENAME) + 2);
		sprintf(fname, "%s/%s", configdir, CONFIG_FILENAME);
	}
	else
	{
		write_stderr("%s does not know where to find the server configuration file.\n"
					 "You must specify the --config-file or -D invocation "
					 "option or set the PGDATA environment variable.\n",
					 progname);
		return false;
	}

	/*
	 * Set the ConfigFileName GUC variable to its final value, ensuring that
	 * it can't be overridden later.
	 */
	SetConfigOption("config_file", fname, PGC_POSTMASTER, PGC_S_OVERRIDE);
	free(fname);

	/*
	 * Now read the config file for the first time.
	 */
	if (stat(ConfigFileName, &stat_buf) != 0)
	{
		write_stderr("%s: could not access the server configuration file \"%s\": %s\n",
					 progname, ConfigFileName, strerror(errno));
		free(configdir);
		return false;
	}

	/*
	 * Read the configuration file for the first time.  This time only the
	 * data_directory parameter is picked up to determine the data directory,
	 * so that we can read the PG_AUTOCONF_FILENAME file next time.
	 */
	ProcessConfigFile(PGC_POSTMASTER);

	/*
	 * If the data_directory GUC variable has been set, use that as DataDir;
	 * otherwise use configdir if set; else punt.
	 *
	 * Note: SetDataDir will copy and absolute-ize its argument, so we don't
	 * have to.
	 */
	if (data_directory)
		SetDataDir(data_directory);
	else if (configdir)
		SetDataDir(configdir);
	else
	{
		write_stderr("%s does not know where to find the database system data.\n"
					 "This can be specified as \"data_directory\" in \"%s\", "
					 "or by the -D invocation option, or by the "
					 "PGDATA environment variable.\n",
					 progname, ConfigFileName);
		return false;
	}

	/*
	 * Reflect the final DataDir value back into the data_directory GUC var.
	 * (If you are wondering why we don't just make them a single variable,
	 * it's because the EXEC_BACKEND case needs DataDir to be transmitted to
	 * child backends specially.  XXX is that still true?  Given that we now
	 * chdir to DataDir, EXEC_BACKEND can read the config file without knowing
	 * DataDir in advance.)
	 */
	SetConfigOption("data_directory", DataDir, PGC_POSTMASTER, PGC_S_OVERRIDE);

	/*
	 * Now read the config file a second time, allowing any settings in the
	 * PG_AUTOCONF_FILENAME file to take effect.  (This is pretty ugly, but
	 * since we have to determine the DataDir before we can find the autoconf
	 * file, the alternatives seem worse.)
	 */
	ProcessConfigFile(PGC_POSTMASTER);

	/*
	 * If timezone_abbreviations wasn't set in the configuration file, install
	 * the default value.  We do it this way because we can't safely install a
	 * "real" value until my_exec_path is set, which may not have happened
	 * when InitializeGUCOptions runs, so the bootstrap default value cannot
	 * be the real desired default.
	 */
	pg_timezone_abbrev_initialize();

	/*
	 * Figure out where pg_hba.conf is, and make sure the path is absolute.
	 */
	if (HbaFileName)
		fname = make_absolute_path(HbaFileName);
	else if (configdir)
	{
		fname = guc_malloc(FATAL,
						   strlen(configdir) + strlen(HBA_FILENAME) + 2);
		sprintf(fname, "%s/%s", configdir, HBA_FILENAME);
	}
	else
	{
		write_stderr("%s does not know where to find the \"hba\" configuration file.\n"
					 "This can be specified as \"hba_file\" in \"%s\", "
					 "or by the -D invocation option, or by the "
					 "PGDATA environment variable.\n",
					 progname, ConfigFileName);
		return false;
	}
	SetConfigOption("hba_file", fname, PGC_POSTMASTER, PGC_S_OVERRIDE);
	free(fname);

	/*
	 * Likewise for pg_ident.conf.
	 */
	if (IdentFileName)
		fname = make_absolute_path(IdentFileName);
	else if (configdir)
	{
		fname = guc_malloc(FATAL,
						   strlen(configdir) + strlen(IDENT_FILENAME) + 2);
		sprintf(fname, "%s/%s", configdir, IDENT_FILENAME);
	}
	else
	{
		write_stderr("%s does not know where to find the \"ident\" configuration file.\n"
					 "This can be specified as \"ident_file\" in \"%s\", "
					 "or by the -D invocation option, or by the "
					 "PGDATA environment variable.\n",
					 progname, ConfigFileName);
		return false;
	}
	SetConfigOption("ident_file", fname, PGC_POSTMASTER, PGC_S_OVERRIDE);
	free(fname);

	free(configdir);

	return true;
}
```
通过上述三个步骤设置完参数后还要检验参数的合法性。比如，数据目录的用户ID应该等于当前进程的有效用户ID、数据目录应该禁止组用户和其他用户的一切访问、缓冲区的数量至少是允许连接的进程数的两倍并且至少为16等。如果一切合法，则将当前目录转入数据目录，然后进行后续的操作。
![alt text](/img/postimg/image2.jpg)
# 2 在pg内核中新增guc参数
在guc.c中定义一个变量保存guc参数
```c
bool        testguc;
```
然后在当前文件中的对应这个类型的数组里面增加对这个参数的定义，这里我们申明的是一个bool类型的guc参数，所以加入到`ConfigNameBool`数组中，如下所示：
```c
	{
		{"testguc", PGC_SIGHUP, UNGROUPED,
			gettext_noop("test guc in pg"),
			NULL,
		},
		&testguc,
		false,
		NULL,NULL,NULL
	},
```
然后在guc.h中extern这个变量就可以了：
```c
extern      bool        testguc;
```
![alt text](/img/postimg/image.png)

# 3 在插件中新增guc参数
首先创建好插件，然后直接在插件对应的c文件中使用`DefineCustomBoolVariable`函数新增一个guc参数，这里我们新增还是一个bool类型的，所以使用的是`DefineCustomBoolVariable`这个函数，如果申请的是其他类型的参数的话就需要使用其他类型的函数去实现，

然后这里需要在`_PG_init`这个函数中去增加这个参数，并且插件中需要增加`PG_MODULE_MAGIC`宏定义。

具体实现如下:
```c
#include "postgres.h"
#include "utils/guc.h"

PG_MODULE_MAGIC;

static bool testgucinextension;


void
_PG_init(void)
{
    DefineCustomBoolVariable("testgucinextension",
	    "test guc in extension",
	    NULL,
	    &testgucinextension,
	    false,
	    PGC_SUSET,
	    false,
	    NULL,
	    NULL,
	    NULL);
}
```
插件对应的Makefile文件内容如下：
```makefile
MODULE_big = testguc
OBJS = \
	$(WIN32RES) \
	testguc.o

# EXTENSION = testguc
PGFILEDESC = "testguc -- test use guc in extension "

ifdef USE_PGXS
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
else
subdir = contrib/testguc
top_builddir = ../..
include $(top_builddir)/src/Makefile.global
include $(top_srcdir)/contrib/contrib-global.mk
endif

SHLIB_LINK += $(filter -lssl -lcrypto -lssleay32 -leay32, $(LIBS))
```