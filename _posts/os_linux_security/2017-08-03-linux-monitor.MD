---
layout: post
title: 输出系统的监控信息
category: linux
comments: false
---

一直想怎么在网页上显示机器的CPU，内存信息等，今天测试了一把，总体的思路有了：

- 首先用程序将机器的性能信息获取到；这是本文的重点。
- 然后可以将信息换成json, 实时写入到一个web服务静态文件里，通过前台不断的发送请求，获取到实时监控信息。

## 获取系统信息

如OS，CPURatio, Memory等。

1.MonitorInfo.java:

    package com.atongmu.demo.monitor;

    /**
     * The monitor information.
     */
    public class MonitorInfo {
        /**
         * 可使用内存.
         */
        private long totalMemory;

        /**
         * 剩余内存.
         */
        private long freeMemory;

        /**
         * 最大可使用内存.
         */
        private long maxMemory;

        /**
         * 操作系统.
         */
        private String osName;

        /**
         * 总的物理内存.
         */
        private long totalMemorySize;

        /**
         * 剩余的物理内存.
         */
        private long freePhysicalMemorySize;

        /**
         * 已使用的物理内存.
         */
        private long usedMemory;

        /**
         * 线程总数.
         */
        private int totalThread;

        /**
         * cpu使用率.
         */
        private double cpuRatio;

        public long getTotalMemory() {
            return totalMemory;
        }

        public void setTotalMemory(long totalMemory) {
            this.totalMemory = totalMemory;
        }

        public long getFreeMemory() {
            return freeMemory;
        }

        public void setFreeMemory(long freeMemory) {
            this.freeMemory = freeMemory;
        }

        public long getMaxMemory() {
            return maxMemory;
        }

        public void setMaxMemory(long maxMemory) {
            this.maxMemory = maxMemory;
        }

        public String getOsName() {
            return osName;
        }

        public void setOsName(String osName) {
            this.osName = osName;
        }

        public long getTotalMemorySize() {
            return totalMemorySize;
        }

        public void setTotalMemorySize(long totalMemorySize) {
            this.totalMemorySize = totalMemorySize;
        }

        public long getFreePhysicalMemorySize() {
            return freePhysicalMemorySize;
        }

        public void setFreePhysicalMemorySize(long freePhysicalMemorySize) {
            this.freePhysicalMemorySize = freePhysicalMemorySize;
        }

        public long getUsedMemory() {
            return usedMemory;
        }

        public void setUsedMemory(long usedMemory) {
            this.usedMemory = usedMemory;
        }

        public int getTotalThread() {
            return totalThread;
        }

        public void setTotalThread(int totalThread) {
            this.totalThread = totalThread;
        }

        public double getCpuRatio() {
            return cpuRatio;
        }

        public void setCpuRatio(double cpuRatio) {
            this.cpuRatio = cpuRatio;
        }

        @Override
        public String toString() {
            final StringBuilder builder = new StringBuilder("MonitorInfo{");
            builder.append("totalMemory=");
            builder.append(totalMemory);
            builder.append(", freeMemory=");
            builder.append(freeMemory);
            builder.append(", maxMemory=");
            builder.append(maxMemory);
            builder.append(", osName=");
            builder.append(osName);
            builder.append(", totalMemorySize=");
            builder.append(totalMemorySize);
            builder.append(", freePhysicalMemorySize=");
            builder.append(freePhysicalMemorySize);
            builder.append(", usedMemory=");
            builder.append(usedMemory);
            builder.append(", totalThread=");
            builder.append(totalThread);
            builder.append(", cpuRatio=");
            builder.append(cpuRatio);
            builder.append('}');
            return builder.toString();
        }
    }

2.MonitorService.java:

    package com.atongmu.demo.monitor;

    import java.io.BufferedReader;
    import java.io.IOException;
    import java.io.InputStreamReader;
    import java.io.LineNumberReader;
    import java.lang.management.ManagementFactory;

    import com.sun.management.OperatingSystemMXBean;

    /**
     * To get the system runtime information.
     */
    public class MonitorService {
        //可以设置长些，防止读到运行此次系统检查时的cpu占用率，就不准了
        private static final int CPU_TIME = 5000;

        private static final int PERCENT = 100;
        private static final int FAULT_LENGTH = 10;
        private static final int KILO_UNIT = 1024;

        public MonitorInfo getMonitorInfo() {
            // 可使用内存 (KB)
            long totalMemory = Runtime.getRuntime().totalMemory() / KILO_UNIT;
            // 剩余内存
            long freeMemory = Runtime.getRuntime().freeMemory() / KILO_UNIT;
            // 最大可使用内存
            long maxMemory = Runtime.getRuntime().maxMemory() / KILO_UNIT;

            OperatingSystemMXBean operatingSystemMXBean = (OperatingSystemMXBean) ManagementFactory
                    .getOperatingSystemMXBean();
            // 操作系统名称
            String osName = System.getProperty("os.name");
            // 总的物理内存
            long totalMemorySize = operatingSystemMXBean.getTotalPhysicalMemorySize() / KILO_UNIT;
            // 剩余的物理内存
            long freePhysicalMemorySize = operatingSystemMXBean.getFreePhysicalMemorySize() / KILO_UNIT;
            // 已使用的物理内存
            long usedMemory = (operatingSystemMXBean.getTotalPhysicalMemorySize() - operatingSystemMXBean
                    .getFreePhysicalMemorySize()) / KILO_UNIT;

            // 获得线程总数
            ThreadGroup parentThread;
            for (parentThread = Thread.currentThread().getThreadGroup(); parentThread
                    .getParent() != null; parentThread = parentThread.getParent()) {
                ;
            }
            int totalThread = parentThread.activeCount();

            double cpuRatio;
            if (osName.toLowerCase().startsWith("windows")) {
                cpuRatio = this.getCpuRatioForWindows();
            } else {
                cpuRatio = operatingSystemMXBean.getSystemCpuLoad();
            }

            // 构造返回对象
            MonitorInfo infoBean = new MonitorInfo();
            infoBean.setFreeMemory(freeMemory);
            infoBean.setFreePhysicalMemorySize(freePhysicalMemorySize);
            infoBean.setMaxMemory(maxMemory);
            infoBean.setOsName(osName);
            infoBean.setTotalMemory(totalMemory);
            infoBean.setTotalMemorySize(totalMemorySize);
            infoBean.setTotalThread(totalThread);
            infoBean.setUsedMemory(usedMemory);
            infoBean.setCpuRatio(cpuRatio);
            return infoBean;
        }

        public String getCmdResult(String cmd) {
            Process process;
            Runtime runtime = Runtime.getRuntime();
            StringBuilder stringBuilder = new StringBuilder();
            try {
                process = runtime.exec(cmd);
                BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
                String line;
                while ((line = reader.readLine()) != null) {
                    stringBuilder.append(line).append("\n");
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
            return stringBuilder.toString();
        }

        /**
         * Get the usage ratio of cpu for windows.
         *
         * @return the cpu ratio.
         */
        private double getCpuRatioForWindows() {
            try {
                String procCmd = System.getenv("windir")
                        + "//system32//wbem//wmic.exe process get Caption,CommandLine,"
                        + "KernelModeTime,ReadOperationCount,ThreadCount,UserModeTime,WriteOperationCount";
                // 取进程信息
                long[] c0 = readCpu(Runtime.getRuntime().exec(procCmd));
                Thread.sleep(CPU_TIME);
                long[] c1 = readCpu(Runtime.getRuntime().exec(procCmd));
                if (c0 != null && c1 != null) {
                    long idleTime = c1[0] - c0[0];
                    long busyTime = c1[1] - c0[1];
                    return (double) (PERCENT * (busyTime) / (busyTime + idleTime));
                } else {
                    return 0.0;
                }
            } catch (Exception ex) {
                ex.printStackTrace();
                return 0.0;
            }
        }

        /**
         * Get cpu info.
         *
         * @param proc the process.
         * @return ret
         */
        private long[] readCpu(final Process proc) {
            long[] retn = new long[2];
            try {
                proc.getOutputStream().close();
                InputStreamReader ir = new InputStreamReader(proc.getInputStream());
                LineNumberReader input = new LineNumberReader(ir);
                String line = input.readLine();
                if (line == null || line.length() < FAULT_LENGTH) {
                    return null;
                }
                int capIdx = line.indexOf("Caption");
                int cmdIdx = line.indexOf("CommandLine");
                int rocIdx = line.indexOf("ReadOperationCount");
                int umtIdx = line.indexOf("UserModeTime");
                int kmtIdx = line.indexOf("KernelModeTime");
                int wocIdx = line.indexOf("WriteOperationCount");
                long idleTime = 0;
                long kernelTime = 0;
                long userTime = 0;
                while ((line = input.readLine()) != null) {
                    if (line.length() < wocIdx) {
                        continue;
                    }
                    // 字段出现顺序：Caption,CommandLine,KernelModeTime,ReadOperationCount,
                    // ThreadCount,UserModeTime,WriteOperation
                    String caption = substring(line, capIdx, cmdIdx - 1)
                            .trim();
                    String cmd = substring(line, cmdIdx, kmtIdx - 1).trim();
                    if (cmd.contains("wmic.exe")) {
                        continue;
                    }
                    // log.info("line="+line);
                    if (caption.equals("System Idle Process")
                            || caption.equals("System")) {
                        idleTime += Long.valueOf(
                                substring(line, kmtIdx, rocIdx - 1).trim());
                        idleTime += Long.valueOf(
                                substring(line, umtIdx, wocIdx - 1).trim());
                        continue;
                    }

                    kernelTime += Long.valueOf(
                            substring(line, kmtIdx, rocIdx - 1).trim());
                    userTime += Long.valueOf(
                            substring(line, umtIdx, wocIdx - 1).trim());
                }
                retn[0] = idleTime;
                retn[1] = kernelTime + userTime;
                return retn;
            } catch (Exception ex) {
                ex.printStackTrace();
            } finally {
                try {
                    proc.getInputStream().close();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            return null;
        }

        private static String substring(String src, int start_idx, int end_idx) {
            byte[] b = src.getBytes();
            StringBuilder tgt = new StringBuilder();
            for (int i = start_idx; i <= end_idx; i++) {
                tgt.append((char) b[i]);
            }
            return tgt.toString();
        }
    }


3.Monitor.java:

    public class Monitor {
        public static void main(String[] args) throws InterruptedException {
            MonitorService service = new MonitorService();
            // System.out.println(service.getCmdResult("ps -A"));
            System.out.println(service.getCmdResult("df"));
            while (true) {
                System.out.println(service.getMonitorInfo());
                Thread.sleep(3000);
            }
        }
    }

输出结果：

    Filesystem    512-blocks      Used Available Capacity  iused   ifree %iused  Mounted on
    /dev/disk0s2   235298960 225213088   9573872    96% 28215634 1196734   96%   /
    devfs                359       359         0   100%      622       0  100%   /dev
    map -hosts             0         0         0   100%        0       0  100%   /net
    map auto_home          0         0         0   100%        0       0  100%   /home
    map -fstab             0         0         0   100%        0       0  100%   /Network/Servers

    MonitorInfo{totalMemory=125952, freeMemory=121931, maxMemory=1864192, osName=Mac OS X, totalMemorySize=8388608, freePhysicalMemorySize=266620, usedMemory=8121988, totalThread=5, cpuRatio=0.0}
    MonitorInfo{totalMemory=125952, freeMemory=121931, maxMemory=1864192, osName=Mac OS X, totalMemorySize=8388608, freePhysicalMemorySize=167120, usedMemory=8221488, totalThread=5, cpuRatio=0.34439834024896265}
    MonitorInfo{totalMemory=125952, freeMemory=121931, maxMemory=1864192, osName=Mac OS X, totalMemorySize=8388608, freePhysicalMemorySize=171168, usedMemory=8217440, totalThread=5, cpuRatio=0.12312811980033278}
    MonitorInfo{totalMemory=125952, freeMemory=121931, maxMemory=1864192, osName=Mac OS X, totalMemorySize=8388608, freePhysicalMemorySize=168696, usedMemory=8219912, totalThread=5, cpuRatio=0.1125}
    MonitorInfo{totalMemory=125952, freeMemory=121931, maxMemory=1864192, osName=Mac OS X, totalMemorySize=8388608, freePhysicalMemorySize=165740, usedMemory=8222868, totalThread=5, cpuRatio=0.13072439633638636}

通过以下的代码，几乎能获取到所有想要的消息，只要你懂相应的命令，剩下的就是字符串解析工作了。

    // Usage: getCmdResult("ps -A")
    public String getCmdResult(String cmd) {
        Process process;
        Runtime runtime = Runtime.getRuntime();
        StringBuilder stringBuilder = new StringBuilder();
        try {
            process = runtime.exec(cmd);
            BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
            String line;
            while ((line = reader.readLine()) != null) {
                stringBuilder.append(line).append("\n");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return stringBuilder.toString();
    }