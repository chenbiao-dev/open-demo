## HotDictReloadThread
```
package org.wltea.analyzer.dic;

import org.apache.logging.log4j.Logger;
import org.elasticsearch.common.io.PathUtils;
import org.wltea.analyzer.help.ESPluginLoggerFactory;

import java.io.FileInputStream;
import java.nio.file.Path;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Properties;

/**
 * [EDIT] 2021-03-28
 * 停用词热更新改造
 */
public class HotDictReloadThread implements Runnable {
    private static final Logger logger = ESPluginLoggerFactory.getLogger(HotDictReloadThread.class.getName());

    private static final String JDBC_RELOAD_FILE = "jdbc-reload.properties";

    private static Properties prop = new Properties();

    static {
        try {
            //Class.forName("com.mysql.jdbc.Driver");
            Class.forName("com.mysql.cj.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            logger.error("error", e);
        }
    }

    @Override
    public void run() {
        while(true) {
            logger.info("[=====start=====]reload hot dict from mysql......");
            logger.info("[=====start=====]从mysql热更新词库......");
            Dictionary.getSingleton().reLoadMainDict();
            logger.info("[======end======]从mysql热更新词库......");
            logger.info("[======end======]reload hot dict from mysql......");
            Object interval = prop.get("jdbc.reload.interval");
            try {
                if(interval == null || interval.toString().isEmpty()) {
                    Thread.sleep(Integer.valueOf(interval.toString()));
                    logger.info("sleep[by prop]:" + interval);
                } else {
                    Thread.sleep(60000);
                    logger.info("sleep:" + 60000);
                }
            } catch (InterruptedException e) {
                logger.error("HotDictReloadThread run error", e);
            }

        }

    }

    /**
     * 从数据库获取停用词
     * @return
     */
    public static List getStopWordDict(String rootPath) {
        List<String> list = new ArrayList();
        Connection conn = null;
        Statement ps = null;
        ResultSet rs = null;
        try {
            Path file = PathUtils.get(rootPath, JDBC_RELOAD_FILE);
            prop.load(new FileInputStream(file.toFile()));
            logger.info("[==========]jdbc-reload.properties");
            for(Object key : prop.keySet()) {
                logger.info("[==========]" + key + "=" + prop.getProperty(String.valueOf(key)));
            }

            logger.info("[==========]query hot dict from mysql, " + prop.getProperty("jdbc.reload.stopword.sql") + "......");

            conn = DriverManager.getConnection(
                    prop.getProperty("jdbc.url"),
                    prop.getProperty("jdbc.user"),
                    prop.getProperty("jdbc.password"));
            ps = conn.createStatement();
            rs = ps.executeQuery(prop.getProperty("jdbc.reload.stopword.sql"));

            while(rs.next()) {
                String theWord = rs.getString("word");
                logger.info("[==========]hot word from mysql: " + theWord);
                list.add(theWord);
            }

        } catch (Exception e) {
            logger.error("HotDictReloadThread error 1", e);
        } finally {
            if(rs != null) {
                try {
                    rs.close();
                } catch (SQLException e) {
                    logger.error("HotDictReloadThread error 2", e);
                }
            }
            if(ps != null) {
                try {
                    ps.close();
                } catch (SQLException e) {
                    logger.error("HotDictReloadThread error 3", e);
                }
            }
            if(conn != null) {
                try {
                    conn.close();
                } catch (SQLException e) {
                    logger.error("HotDictReloadThread error 4", e);
                }
            }
        }
        return list;

    }

    /**
     * 从数据库获取更新词
     * @return
     */
    public static List getMainDict(String rootPath) {
        List<String> list = new ArrayList();
        Connection conn = null;
        Statement ps = null;
        ResultSet rs = null;
        try {
            Path file = PathUtils.get(rootPath, JDBC_RELOAD_FILE);
            prop.load(new FileInputStream(file.toFile()));
            logger.info("[==========]jdbc-reload.properties");
            for(Object key : prop.keySet()) {
                logger.info("[==========]" + key + "=" + prop.getProperty(String.valueOf(key)));
            }

            logger.info("[==========]query hot dict from mysql, " + prop.getProperty("jdbc.reload.sql") + "......");

            conn = DriverManager.getConnection(
                    prop.getProperty("jdbc.url"),
                    prop.getProperty("jdbc.user"),
                    prop.getProperty("jdbc.password"));
            ps = conn.createStatement();
            rs = ps.executeQuery(prop.getProperty("jdbc.reload.sql"));

            while(rs.next()) {
                String theWord = rs.getString("word");
                logger.info("[==========]hot word from mysql: " + theWord);
                list.add(theWord);
            }

        } catch (Exception e) {
            logger.error("HotDictReloadThread error 5", e);
        } finally {
            if(rs != null) {
                try {
                    rs.close();
                } catch (SQLException e) {
                    logger.error("HotDictReloadThread error 6", e);
                }
            }
            if(ps != null) {
                try {
                    ps.close();
                } catch (SQLException e) {
                    logger.error("HotDictReloadThread error 7", e);
                }
            }
            if(conn != null) {
                try {
                    conn.close();
                } catch (SQLException e) {
                    logger.error("HotDictReloadThread error 8", e);
                }
            }
        }
        return list;
    }
}

```

## Dictionary.java

```
Dictionary.class
#initial
public static synchronized void initial(Configuration cfg) {
		if (singleton == null) {
			synchronized (Dictionary.class) {
				if (singleton == null) {

					singleton = new Dictionary(cfg);
					singleton.loadMainDict();
					singleton.loadSurnameDict();
					singleton.loadQuantifierDict();
					singleton.loadSuffixDict();
					singleton.loadPrepDict();
					singleton.loadStopWordDict();

					// [EDIT] 2021-03-28
					// Step1.开启新的线程重新加载词典
					new Thread(new HotDictReloadThread()).start();

					if(cfg.isEnableRemoteDict()){
						// 建立监控线程
						for (String location : singleton.getRemoteExtDictionarys()) {
							// 10 秒是初始延迟可以修改的 60是间隔时间 单位秒
							pool.scheduleAtFixedRate(new Monitor(location), 10, 60, TimeUnit.SECONDS);
						}
						for (String location : singleton.getRemoteExtStopWordDictionarys()) {
							pool.scheduleAtFixedRate(new Monitor(location), 10, 60, TimeUnit.SECONDS);
						}
					}

				}
			}
		}
	}

#reLoadMainDict
void reLoadMainDict() {
		logger.info("start to reload ik dict.");
		// 新开一个实例加载词典，减少加载过程对当前词典使用的影响
		Dictionary tmpDict = new Dictionary(configuration);
		tmpDict.configuration = getSingleton().configuration;
		tmpDict.loadMainDict();
		tmpDict.loadStopWordDict();
		// [EDIT] 2021-03-28
		// 从mysql加载主词典
		tmpDict.reLoadMainDict();
		_MainDict = tmpDict._MainDict;
		_StopWords = tmpDict._StopWords;
		logger.info("reload ik dict finished.");
	}

#reLoadMySqlExtDict 新增
    // [EDIT] 2021-03-28
	// 从mysql加载主词典
	void reLoadMySqlExtDict() {
		// 主词库重新加载
		List<String> mainDictList = HotDictReloadThread.getMainDict(getDictRoot());
		for (String theWord : mainDictList) {
			if (theWord != null && !"".equals(theWord.trim())) {
				// 加载扩展词典数据到主内存词典中
				logger.info(theWord);
				_MainDict.fillSegment(theWord.trim().toLowerCase().toCharArray());
			}
		}

		// 停用词重新加载
		List<String> stopWordDictList = HotDictReloadThread.getStopWordDict(getDictRoot());
		for (String theWord : stopWordDictList) {
			if (theWord != null && !"".equals(theWord.trim())) {
				// 加载远程词典数据到主内存中
				logger.info(theWord);
				_StopWords.fillSegment(theWord.trim().toLowerCase().toCharArray());
			}
		}

	}

```
## config文件夹下新建 jdbc-reload.properties
```
jdbc.url=jdbc:mysql://localhost:3306/ik?serverTimezone=GMT
jdbc.user=root
jdbc.password=root
jdbc.reload.sql=select word from hot_words
jdbc.reload.stopword.sql=select stopword as word from hot_stopwords
# 60 秒轮询
jdbc.reload.interval=60000   


```