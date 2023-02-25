### jsch
- dependency

```
 <!-- sftp上传依赖包 -->
        <dependency>
            <groupId>com.jcraft</groupId>
            <artifactId>jsch</artifactId>
            <version>0.1.55</version>
        </dependency>
```

- code
```
package com.aaa.sftp;
 
import com.jcraft.jsch.Channel;
import com.jcraft.jsch.ChannelSftp;
import com.jcraft.jsch.JSch;
import com.jcraft.jsch.JSchException;
import com.jcraft.jsch.Session;
import com.jcraft.jsch.SftpException;

import org.apache.commons.io.IOUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayInputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.UnsupportedEncodingException;
import java.util.Properties;
import java.util.Vector;
/**
 * 
 * @ClassName: SFTPUtil
 * @Description: sftp连接工具类
 * @date 2017年5月22日 下午11:17:21
 * @version 1.0.0
 */
public class SFTPUtil {
	private transient Logger log = LoggerFactory.getLogger(this.getClass());
    
    private ChannelSftp sftp;
      
    private Session session;
    /** FTP 登录用户名*/  
    private String username;
    /** FTP 登录密码*/  
    private String password;
    /** 私钥 */  
    private String privateKey;
    /** FTP 服务器地址IP地址*/  
    private String host;
    /** FTP 端口*/
    private int port;
      
  
    /** 
     * 构造基于密码认证的sftp对象 
     * @param userName 
     * @param password 
     * @param host 
     * @param port 
     */  
    public SFTPUtil(String username, String password, String host, int port) {
        this.username = username;
        this.password = password;
        this.host = host;
        this.port = port;
    }
  
    /** 
     * 构造基于秘钥认证的sftp对象
     * @param userName
     * @param host
     * @param port
     * @param privateKey
     */
    public SFTPUtil(String username, String host, int port, String privateKey) {
        this.username = username;
        this.host = host;
        this.port = port;
        this.privateKey = privateKey;
    }
  
    public SFTPUtil(){}
  
    /**
     * 连接sftp服务器
     *
     * @throws Exception 
     */
    public void login(){
        try {
            JSch jsch = new JSch();
            if (privateKey != null) {
                jsch.addIdentity(privateKey);// 设置私钥
                log.info("sftp connect,path of private key file：{}" , privateKey);
            }
            log.info("sftp connect by host:{} username:{}",host,username);
  
            session = jsch.getSession(username, host, port);
            log.info("Session is build");
            if (password != null) {
                session.setPassword(password);  
            }
            Properties config = new Properties();
            config.put("StrictHostKeyChecking", "no");
              
            session.setConfig(config);
            session.connect();
            log.info("Session is connected");
            
            Channel channel = session.openChannel("sftp");
            channel.connect();
            log.info("channel is connected");
  
            sftp = (ChannelSftp) channel;
            log.info(String.format("sftp server host:[%s] port:[%s] is connect successfull", host, port));
        } catch (JSchException e) {
            log.error("Cannot connect to specified sftp server : {}:{} \n Exception message is: {}",
                new Object[]{host, port, e.getMessage()});
        }
    }  
  
    /**
     * 关闭连接 server 
     */
    public void logout(){
        if (sftp != null) {
            if (sftp.isConnected()) {
                sftp.disconnect();
                log.info("sftp is closed already");
            }
        }
        if (session != null) {
            if (session.isConnected()) {
                session.disconnect();
                log.info("sshSession is closed already");
            }
        }
    }
  
    /** 
     * 将输入流的数据上传到sftp作为文件 
     *  
     * @param directory 
     *            上传到该目录 
     * @param sftpFileName 
     *            sftp端文件名 
     * @param in 
     *            输入流 
     * @throws SftpException  
     * @throws Exception 
     */  
    public void upload(String directory, String sftpFileName, InputStream input) throws SftpException{
        try {  
            sftp.cd(directory);
        } catch (SftpException e) {
            log.warn("directory is not exist");
            sftp.mkdir(directory);
            sftp.cd(directory);
        }
        sftp.put(input, sftpFileName);
        log.info("file:{} is upload successful" , sftpFileName);
    }
  
    /** 
     * 上传单个文件
     *
     * @param directory 
     *            上传到sftp目录 
     * @param uploadFile
     *            要上传的文件,包括路径 
     * @throws FileNotFoundException
     * @throws SftpException
     * @throws Exception
     */
    public void upload(String directory, String uploadFile) throws FileNotFoundException, SftpException{
        File file = new File(uploadFile);
        upload(directory, file.getName(), new FileInputStream(file));
    }
  
    /**
     * 将byte[]上传到sftp，作为文件。注意:从String生成byte[]是，要指定字符集。
     * 
     * @param directory
     *            上传到sftp目录
     * @param sftpFileName
     *            文件在sftp端的命名
     * @param byteArr
     *            要上传的字节数组
     * @throws SftpException
     * @throws Exception
     */
    public void upload(String directory, String sftpFileName, byte[] byteArr) throws SftpException{
        upload(directory, sftpFileName, new ByteArrayInputStream(byteArr));
    }
  
    /** 
     * 将字符串按照指定的字符编码上传到sftp
     *  
     * @param directory
     *            上传到sftp目录
     * @param sftpFileName
     *            文件在sftp端的命名
     * @param dataStr
     *            待上传的数据
     * @param charsetName
     *            sftp上的文件，按该字符编码保存
     * @throws UnsupportedEncodingException
     * @throws SftpException
     * @throws Exception
     */
    public void upload(String directory, String sftpFileName, String dataStr, String charsetName) throws UnsupportedEncodingException, SftpException{  
        upload(directory, sftpFileName, new ByteArrayInputStream(dataStr.getBytes(charsetName)));  
    }
  
    /**
     * 下载文件 
     *
     * @param directory
     *            下载目录 
     * @param downloadFile
     *            下载的文件
     * @param saveFile
     *            存在本地的路径
     * @throws SftpException
     * @throws FileNotFoundException
     * @throws Exception
     */  
    public void download(String directory, String downloadFile, String saveFile) throws SftpException, FileNotFoundException{
        if (directory != null && !"".equals(directory)) {
            sftp.cd(directory);
        }
        File file = new File(saveFile);
        sftp.get(downloadFile, new FileOutputStream(file));
        log.info("file:{} is download successful" , downloadFile);
    }
    /** 
     * 下载文件
     * @param directory 下载目录
     * @param downloadFile 下载的文件名
     * @return 字节数组
     * @throws SftpException
     * @throws IOException
     * @throws Exception
     */
    public byte[] download(String directory, String downloadFile) throws SftpException, IOException{
        if (directory != null && !"".equals(directory)) {
            sftp.cd(directory);
        }
        InputStream is = sftp.get(downloadFile);
        
        byte[] fileData = IOUtils.toByteArray(is);
        
        log.info("file:{} is download successful" , downloadFile);
        return fileData;
    }
  
    /**
     * 删除文件
     *  
     * @param directory
     *            要删除文件所在目录
     * @param deleteFile
     *            要删除的文件
     * @throws SftpException
     * @throws Exception
     */
    public void delete(String directory, String deleteFile) throws SftpException{
        sftp.cd(directory);
        sftp.rm(deleteFile);
    }
  
    /**
     * 列出目录下的文件
     * 
     * @param directory
     *            要列出的目录
     * @param sftp
     * @return
     * @throws SftpException
     */
    public Vector<?> listFiles(String directory) throws SftpException {
        return sftp.ls(directory);
    }
    
    public static void main(String[] args) throws SftpException, IOException {
        SFTPUtil sftp = new SFTPUtil(
            "FILEGW_SFTP_INSIDE_PWD",
            "hauwei@123",
            "127.0.0.1",
            7022);
        sftp.login();
        //byte[] buff = sftp.download("/opt", "start.sh");
        //System.out.println(Arrays.toString(buff));
        File file = new File(
            "D:\\data01\\filegw\\data\\mount\\files\\com.huawei.roma.filegw\\http_b2b\\testdmz.txt");
        InputStream is = new FileInputStream(file);
        
        sftp.upload("/20221209-b2b-sftp", "testdmz.txt", is);
        sftp.logout();
    }
}
```

### sshd

- dependency
```
<dependency>
            <groupId>org.apache.sshd</groupId>
            <artifactId>sshd-core</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.sshd</groupId>
            <artifactId>sshd-sftp</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.sshd</groupId>
            <artifactId>sshd-common</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.sshd</groupId>
            <artifactId>sshd-netty</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.sshd</groupId>
            <artifactId>sshd-scp</artifactId>
            <version>2.8.0</version>
        </dependency>
```

- code

```

package com.aaa.sftp;

import org.apache.sshd.client.SshClient;
import org.apache.sshd.client.channel.ChannelExec;
import org.apache.sshd.client.session.ClientSession;
import org.apache.sshd.scp.client.DefaultScpClientCreator;
import org.apache.sshd.scp.client.ScpClient;
import org.apache.sshd.scp.client.ScpClientCreator;
import org.apache.sshd.sftp.client.SftpClient;
import org.apache.sshd.sftp.client.impl.DefaultSftpClientFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;


/**
 * 
 * @author wangzonghui
 * @date 2022年2月7日 上午9:55:58 
 * @Description ssh工具类测试
 */
public final class SshSshdUtil {
	
	private  final static Logger log = LoggerFactory.getLogger(SshSshdUtil.class);
	
	private String host;
	private String user;
	private String password;
	private int port;
	
	 private ClientSession session;
	 private  SshClient client;
	
	/**
	 * 创建一个连接
	 * 
	 * @param host     地址
	 * @param user     用户名
	 * @param port     ssh端口
	 * @param password 密码
	 */
	public SshSshdUtil(String host,String user,int port,String password) {
		this.host = host;
		this.user = user;
		this.port=port;
		this.password=password;
	}

	/**
	 * 登录
	 * @return
	 * @throws Exception
	 */
	public boolean initialSession() {
		if (session == null) {
			
			try {
				// 创建 SSH客户端
				client = SshClient.setUpDefaultClient();
				// 启动 SSH客户端
				client.start();
				// 通过主机IP、端口和用户名，连接主机，获取Session
				session = client.connect(user, host, port).verify().getSession();
				// 给Session添加密码
				session.addPasswordIdentity(password);
				// 校验用户名和密码的有效性
				return session.auth().verify().isSuccess();
				
			} catch (Exception e) {
				log.info("Login Host:"+host+" Error",e);
				return false;
			}
		}
		
		return true;
	}

	/**
	 * 关闭连接
	 * @throws Exception
	 */
	public void close() throws Exception  {
		//关闭session
		if (session != null && session.isOpen()) {
            session.close();
        }
		
		// 关闭 SSH客户端
        if (client != null && client.isOpen()) {
            client.stop();
            client.close();
        }
	}
	
	/**
	 * 下载文件 基于sftp
	 * 
	 * @param localFile  本地文件名，若为空或是*，表示目前下全部文件
	 * @param remotePath 远程路径，若为空，表示当前路径，若服务器上无此目录，则会自动创建
	 * @throws Exception
	 */
	public boolean sftpGetFile(String localFile, String remoteFile) {
		SftpClient sftp=null;
		InputStream is=null;
		try {
			if(this.initialSession()) {
				DefaultSftpClientFactory sftpFactory=new DefaultSftpClientFactory();
				sftp=sftpFactory.createSftpClient(session);
				is=sftp.read(remoteFile);
		        Path dst=Paths.get(localFile);
		        Files.deleteIfExists(dst);
		        Files.copy(is, dst);
		        
			}
		} catch (Exception e) {
			log.error(host+"Local File "+localFile+" Sftp Get File:"+remoteFile+" Error",e);
			return false;
		}finally {
			try {
				if(is!=null) {
					is.close();
				}
				if(sftp!=null) {
					sftp.close();
				}
				
			} catch (Exception e) {
				log.error("Close Error",e);
			}
			
		}
		return true;
	}
	
	/**
	 * 下载文件 基于sftp
	 * 
	 * @param localFile  本地文件名，若为空或是*，表示目前下全部文件
	 * @param remotePath 远程路径，若为空，表示当前路径，若服务器上无此目录，则会自动创建
	 * @param outTime 超时时间 单位毫秒
	 * @throws Exception
	 */
	public boolean sftpGetFile(String localFile, String remoteFile,int timeOut) {
		ProcessWithTimeout process=new ProcessWithTimeout(localFile,remoteFile,1);
		int exitCode=process.waitForProcess(timeOut);
		
		if(exitCode==-1) {
			log.error("{} put Local File {} To Sftp Path:{} Time Out",host,localFile,remoteFile);
		}
		return exitCode==0?true:false;
	}

	/**
	 * 上传文件 基于sftp
	 * 
	 * @param localFile  本地文件名，若为空或是*，表示目前下全部文件
	 * @param remotePath 远程路径，若为空，表示当前路径，若服务器上无此目录，则会自动创建
	 * @param outTime 超时时间 单位毫秒
	 * @throws Exception
	 */
	public boolean sftpPutFile(String localFile, String remoteFile,int timeOut) {
		ProcessWithTimeout process=new ProcessWithTimeout(localFile,remoteFile,0);
		int exitCode=process.waitForProcess(timeOut);
		
		if(exitCode==-1) {
			log.error("{} put Local File {} To Sftp Path:{} Time Out",host,localFile,remoteFile);
		}
		return exitCode==0?true:false;
	}
	
	/**
	 * 
	 * @author wangzonghui
	 * @date 2022年4月13日 下午2:15:17 
	 * @Description 任务执行线程，判断操作超时使用
	 */
	class ProcessWithTimeout extends Thread{
		
		private String localFile;
		private String remoteFile;
		private int type;
		private int exitCode =-1;
		
		/**
		 * 
		 * @param localFile 本地文件
		 * @param remoteFile  sftp服务器文件
		 * @param type 0 上传  1 下载
		 */
		public ProcessWithTimeout(String localFile, String remoteFile,int type) {
			this.localFile=localFile;
			this.remoteFile=remoteFile;
			this.type=type;
		}
		
		public int waitForProcess(int outtime){
			  this.start();
			  
			  try{
			     this.join(outtime);
			  }catch (InterruptedException e){
			   log.error("Wait Is Error",e);
			  }
			  
			  return exitCode;
		 }
		
		@Override
		public void run() {
			super.run();
			boolean state;
			if(type==0) {
				state=sftpPutFile(localFile, remoteFile);
			}else {
				state=sftpGetFile(localFile, remoteFile);
			}
			
			exitCode= state==true?0:1;
		}
	}

	public static void main(String[] args) {
		SshSshdUtil sshSshdUtil = new SshSshdUtil(
			"127.0.0.1",
			"filegw_facc",
			7022,
			"fjh@HWW411501"
		);
		sshSshdUtil.sftpPutFile(
			"D:\\data01\\filegw\\data\\mount\\files\\com.huawei.roma.filegw\\http_b2b\\testdmz.txt",
			"/20221209-b2b-sftp/testdmz.txt");
	}
	/**
	 * 上传文件 基于sftp
	 * 
	 * @param localFile  本地文件名，若为空或是*，表示目前下全部文件
	 * @param remotePath 远程路径，若为空，表示当前路径，若服务器上无此目录，则会自动创建
	 * @throws Exception
	 */
	public boolean sftpPutFile(String localFile, String remoteFile) {
		SftpClient sftp=null;
		OutputStream os=null;
		try {
			if(this.initialSession()) {
				DefaultSftpClientFactory sftpFactory=new DefaultSftpClientFactory();
				sftp=sftpFactory.createSftpClient(session);

				os=sftp.write(remoteFile,1024);
		        Files.copy(Paths.get(localFile), os);
			}

		} catch (Exception e) {
			log.error(host+"Local File "+localFile+" Sftp Upload File:"+remoteFile+" Error",e);
			return false;
		}finally {
			try {
				if(os!=null) {
					os.close();
				}
				if(sftp!=null) {
					sftp.close();
				}
				
			} catch (Exception e) {
				log.error("Close Error",e);
			}
		}
		return true;
	}

	/**
	 * 上传文件 基于scp
	 * 
	 * @param localFile  本地文件名，若为空或是*，表示目前下全部文件
	 * @param remotePath 远程路径，若为空，表示当前路径，若服务器上无此目录，则会自动创建
	 * @throws Exception
	 */
	public boolean scpPutFile( String localFile, String remoteFile) {
		ScpClient scpClient=null;
		try {
			if(this.initialSession()) {
				ScpClientCreator creator = new DefaultScpClientCreator();

	            // 创建 SCP 客户端
	            scpClient = creator.createScpClient(session);

	            // ScpClient.Option.Recursive：递归copy，可以将子文件夹和子文件遍历copy
	            scpClient.upload(localFile, remoteFile, ScpClient.Option.Recursive);
	            
			}else {
				log.error("Host:{} User:{} Upload Local File:{} Error",host,user,localFile);
				return false;
			}
			
		} catch (Exception e) {
			log.error(e.toString(), e);
			return false;
		}finally {
			// 释放 SCP客户端
            if (scpClient != null) {
                scpClient = null;
            }
		}

		return true;
	}

	/**
	 * 下载文件 基于scp
	 * 
	 * @param localPath  本地路径，若为空，表示当前路径
	 * @param localFile  本地文件名，若为空或是*，表示目前下全部文件
	 * @param remotePath 远程路径，若为空，表示当前路径，若服务器上无此目录，则会自动创建
	 * @throws Exception
	 */
	public boolean scpGetFile( String localFile, String remoteFile) {
		ScpClient scpClient=null;
		try {
			if(this.initialSession()) {
				ScpClientCreator creator = new DefaultScpClientCreator();
	            // 创建 SCP 客户端
	            scpClient = creator.createScpClient(session);

	            scpClient.download(remoteFile, localFile);  //下载文件
	            
			}else {
				log.error("Host:{} User:{} Get File:{} Error",host,user,remoteFile);
				return false;
			}
			
		} catch (Exception e) {
			log.error(e.toString(), e);
			return false;
		}finally {
			 // 释放 SCP客户端
            if (scpClient != null) {
                scpClient = null;
            }
            
		}

		return true;
	}
	
	/**
	 * 执行远程命令 
	 * @param command 执行的命令
	 * @return 0成功 1异常
	 * @throws Exception
	 */
	public int runCommand(String command)  {
		ChannelExec channel=null;
		try {
			if(this.initialSession()) {
				channel=session.createExecChannel(command);
				int time = 0;
				boolean run = false;
				
				channel.open();
				ByteArrayOutputStream err = new ByteArrayOutputStream();
				channel.setErr(err);
				while (true) {
					if (channel.isClosed() || run) {
						break;
					}
					try {
						Thread.sleep(100);
					} catch (Exception e) {
						
					}
					if (time > 1800) {
						break;
					}
					time++;
				}
				int status=channel.getExitStatus();
				if(status>0) {
					log.info("{}  host:{} user:{} Run Is code:{} Message:{}",command,host,user,status,err.toString());
				}
				return status;
			}else {
				log.error("Host:{} User:{} Login Error",host,user);
				return 1;
			}
			
		} catch (Exception e) {
			log.error("Host "+host+" Run Command Error:"+command+" " +e.toString(),e);
			return 1;
		}finally {
			if(channel!=null) {
				try {
					channel.close();
				} catch (Exception e) {
					log.error("Close Connection Error");
				}
			}
			
		}
	}

	// /**
	//  * 执行远程命令
	//  * @param command 执行的命令
	//  * @return 0成功 其他 异常
	//  * @throws Exception
	//  */
	// public SshModel run(String command)  {
	// 	SshModel sshModel=new SshModel();
	// 	ChannelExec channel=null;
	// 	try {
	// 		if(this.initialSession()) {
	// 			channel=session.createExecChannel(command);
	// 			int time = 0;
	// 			boolean run = false;
	//
	// 			channel.open();
	// 			ByteArrayOutputStream info = new ByteArrayOutputStream();
	// 			channel.setOut(info);
	// 			ByteArrayOutputStream err = new ByteArrayOutputStream();
	// 			channel.setErr(err);
	//
	// 			while (true) {
	// 				if (channel.isClosed() || run) {
	// 					break;
	// 				}
	// 				try {
	// 					Thread.sleep(100);
	// 				} catch (Exception ee) {
	// 				}
	// 				if (time > 180) {
	//
	// 					break;
	// 				}
	// 				time++;
	// 			}
	//
	// 	        int status=channel.getExitStatus();
	//
	// 			sshModel.setCode(status);
	// 			sshModel.setInfo(info.toString());
	// 			sshModel.setError(err.toString());
	// 		}else {
	// 			log.error("Host:{} User:{} Login Error",host,user);
	// 			sshModel.setCode(1);
	// 			sshModel.setError("Loging Error");
	// 		}
	//
	// 	} catch (Exception e) {
	// 		log.error("Host "+host+"Run Command Error:"+command+" " +e.toString(),e);
	// 		sshModel.setCode(1);
	//
	// 	}finally {
	// 		if(channel!=null) {
	// 			try {
	// 				channel.close();
	// 			} catch (IOException e) {
	// 				log.error("Close Connection Error");
	// 			}
	// 		}
	// 	}
	// 	return sshModel;
	// }
}
```