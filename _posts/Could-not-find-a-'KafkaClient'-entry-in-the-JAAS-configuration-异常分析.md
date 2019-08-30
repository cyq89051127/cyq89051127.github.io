---
layout: post
title:  "Could-not-find-a-'KafkaClient'-entry-in-the-JAAS-configuration-异常分析"
date:   2018-12-30 21:38:12 +0800
tags:
      - Kafka
---

Kafka提供了KafkaProducer和KakfaConsumer用于生产和消费数据。0.9之后的kafka集群基于kerberos实现了安全的kafka访问机制。然而在kafka访问时，需要的principal，keytab，ServiceName等信息均配置在jaas文件中，因此保证认证的服务可以读取到正确的文件及正确的配置是kafka安全模式下认证的核心。

本文对安全模式下的kafka集群访问时，如何查找认证需要的信息进行分析。

![kafka_jaas读取流程.jpg](https://upload-images.jianshu.io/upload_images/9004616-3025bcc0a222aca0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

查看如上流程的代码逻辑可以看出，kakfa认证的核心在于jaas文件的查找与解析。

         AppConfigurationEntry[] configurationEntries = Configuration.getConfiguration().getAppConfigurationEntry(loginContextName);
         // 只有在无法获取configurationEntries时，会抛出上述异常。
         if (configurationEntries == null) {
            String errorMessage = "Could not find a '" + loginContextName + "' entry in this configuration.";
            throw new IOException(errorMessage);
        }
此处使用的Configuration为javax.security.auth.login.Configuration.查看其getConfiguration的逻辑可以看出此处获取的Config为ConfigFile.java。其核心逻辑如下：

    //如下，首先获取login.configuration.provider中配置的类（在常用的hadoop安全集群其配置的值为sun.security.provider.ConfigFile），如果没有，则使用默认的sun.security.provider.ConfigFile类，直接反射生成实例。
    synchronized (Configuration.class) {
        if (configuration == null) {
            String config_class = null;
            config_class = AccessController.doPrivileged
                (new PrivilegedAction<String>() {
                public String run() {
                    return java.security.Security.getProperty("login.configuration.provider"); }});
            if (config_class == null) {
                config_class = "sun.security.provider.ConfigFile";
            }
            try { final String finalClass = config_class;
            Configuration untrustedImpl = AccessController.doPrivileged(
                    new PrivilegedExceptionAction<Configuration>() {
                        public Configuration run() throws ClassNotFoundException,
                                InstantiationException,
                                IllegalAccessException {
                            Class<? extends Configuration> implClass = Class.forName( finalClass,false,Thread.currentThread().getContextClassLoader()
                            ).asSubclass(Configuration.class);
                            return implClass.newInstance();
                        }
                    });

在获取ConfigFile之后，调用其getAppConfigurationEntry方法获取需要的信息,该方法通过调用内部类spi直接调用spi.engineGetAppConfigurationEntry实现：

    // 此处的applicationName即是KafkaClient，且不可配置
    public AppConfigurationEntry[] engineGetAppConfigurationEntry
            (String applicationName) {
            List<AppConfigurationEntry> list = null;
            synchronized (configuration) {
            // 直接从configuration中获取相关appConfigEntries
                list = configuration.get(applicationName);
            }
            if (list == null || list.size() == 0) {
                return null;
            }
            AppConfigurationEntry[] entries =
                                    new AppConfigurationEntry[list.size()];
            Iterator<AppConfigurationEntry> iterator = list.iterator();
            for (int i = 0; iterator.hasNext(); i++) {
                AppConfigurationEntry e = iterator.next();
                entries[i] = new AppConfigurationEntry(e.getLoginModuleName(),
                                                       e.getControlFlag(),
                                                       e.getOptions());
            }
            return entries;
        }
那么问题来了， 此处的configuration是如何生成的呢？ 

答案是在ConfigFile初始化时完成configuration的生成，顺序如下：
![configuration初始化流程.jpg](https://upload-images.jianshu.io/upload_images/9004616-e84d6b22a32f9106.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在init()方法中，生成Configuration
    


        /**
         * Read and initialize the entire login Configuration from the
         * configured URL.
         *
         * @throws IOException if the Configuration can not be initialized
         * @throws SecurityException if the caller does not have permission
         *                           to initialize the Configuration
         */
        private void init() throws IOException {

            boolean initialized = false;

            // For policy.expandProperties, check if either a security or system
            // property is set to false (old code erroneously checked the system
            // prop so we must check both to preserve compatibility).
            String expand = Security.getProperty("policy.expandProperties");
            if (expand == null) {
                expand = System.getProperty("policy.expandProperties");
            }
            if ("false".equals(expand)) {
                expandProp = false;
            }

            // new configuration
            Map<String, List<AppConfigurationEntry>> newConfig = new HashMap<>();
            // 此时url为空
            if (url != null) {
                ....
            }

            /**
             * Caller did not specify URI via Configuration.getInstance.
             * Read from URLs listed in the java.security properties file.
             */
            String allowSys = Security.getProperty("policy.allowSystemProperty");
            // 在安全模式下，policy.allowSystemProperty需要设置为true
            if ("true".equalsIgnoreCase(allowSys)) {
            // 此时读取我们设置的-Djava.security.auth.login.config=/path/to/yourjaasFile
                String extra_config = System.getProperty("java.security.auth.login.config");
                if (extra_config != null) {
                    boolean overrideAll = false;
                    if (extra_config.startsWith("=")) {
                        overrideAll = true;
                        extra_config = extra_config.substring(1);
                    }
                    try {
                        extra_config = PropertyExpander.expand(extra_config);
                    } catch (PropertyExpander.ExpandException peee) {
                        throw ioException("Unable.to.properly.expand.config",
                                          extra_config);
                    }

                    URL configURL = null;
                    try {
                        configURL = new URL(extra_config);
                    } catch (MalformedURLException mue) {
                        File configFile = new File(extra_config);
                        if (configFile.exists()) {
                            configURL = configFile.toURI().toURL();
                        } 
                        ...
                    }...
                    //此处调用init完成配置读取
                    init(configURL, newConfig);
                    initialized = true;
                    if (overrideAll) {
                        configuration = newConfig;
                        return;}
                }
            }

            int n = 1;
            String config_url;
            // 获取配置项login.config.url.[1-n]中设置的
            while ((config_url = Security.getProperty
                                     ("login.config.url."+n)) != null) {
                try {config_url = PropertyExpander.expand
                        (config_url).replace(File.separatorChar, '/');
                    if (debugConfig != null) {
                        debugConfig.println("\tReading config: " + config_url);
                    }
                    //调用init完成配置读取
                    init(new URL(config_url), newConfig);
                    initialized = true;
                } ...
                n++;
            }

            if (initialized == false && n == 1 && config_url == null) {...}

            configuration = newConfig;
        }

        private void init(URL config,
                          Map<String, List<AppConfigurationEntry>> newConfig)
                          throws IOException {

            try (InputStreamReader isr = new InputStreamReader(getInputStream(config), "UTF-8")) {
             // 读取配置文件
                readConfig(isr, newConfig);
            }...
            
在readConfig的方法的文件解析方法中中我们可以看到jaas文件的配置要求

总结起来就是kafka认证所需的信息通常在jaas文件中配置，通过-Djava.security.auth.login.config来指定jaas文件的全路径，结合jdk的一些安全配置完成jaas认证所需信息的读取。从代码中可以看出，除了应用配置jaas之外，也可以在jdk的安全配置文件中配置相关文件，但不建议这么使用。因为如果在jdk中配置， 则所有进程都将读取到相关信息。



PS ： 
代码中用到的Security.getProperty时，读取的配置项是在${JAVA_HOME}/jre/lib/security/java.seucurity配置文件中配置。
