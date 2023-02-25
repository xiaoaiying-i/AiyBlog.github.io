### sftp服务器运行流程
- 配置sshd  
```
// SftpServerServiceImpl.class
@PostConstruct
@Override
public void start() throws IOException, FgwException {
    if (server != null && server.isStarted()) {
        return;
    }
    server = SshServer.setUpDefaultServer();
    Map<String, Object> props = server.getProperties();
    PropertyResolverUtils.updateProperty(props, CoreModuleProperties.IDLE_TIMEOUT.getName(), TimeUnit.MINUTES.toMillis(IDLE_TIMEOUT));
    server.setHost(host);
    server.setPort(port);
    server.setKeyPairProvider(new FgwGeneratorHostKeyProvider(hostKeyPath));
    if(s3StoreEnable){
        fgwFileSystemFactory.s3StoreEnable(s3StoreEnable, getS3Config());
    }
    server.setFileSystemFactory(fgwFileSystemFactory);
    server.addSessionListener(sftpSessionListener);
    SftpSubsystemFactory factory = new SftpSubsystemFactory();
    factory.addSftpEventListener(sftpEventHandler);
    server.setSubsystemFactories(Collections.singletonList(factory));
    server.setPasswordAuthenticator(passwordAuthenticator);
    server.setPublickeyAuthenticator(sftpPublicKeyAuthenticator);
    server.start();
}
```

- FileSystemFactory (获取FileSystem) 
```
FileSystemFactory 
private FileSystem getFileSystemProvider(SessionContext session) {
    FileSystemProvider fileSystemProvider;
    if (this.s3StoreEnable) {
        fileSystemProvider = new FgwFileSystemProvider(session, fileSystemService, this.hisS3Config);
        return fileSystemProvider.getFileSystem(URI.create("sftps3:///"));
    }
    fileSystemProvider = new FgwFileSystemProvider(session, fileSystemService);
    return fileSystemProvider.getFileSystem(URI.create("file:///"));
}
```

- FileSystem(需要绑定FileSystemProvider) 
```
// 文件系统抽象类
public abstract class BaseFileSystem<T extends Path> extends FileSystem {
    protected final Logger log = LoggerFactory.getLogger(this.getClass());
    private final FileSystemProvider fileSystemProvider;

    public BaseFileSystem(FileSystemProvider fileSystemProvider) {
        this.fileSystemProvider = (FileSystemProvider)Objects.requireNonNull(fileSystemProvider, "No file system provider");
    }

// fgw文件系统
public FgwFileSystem(FileSystemProvider fileSystemProvider, SessionContext session) {
    super(fileSystemProvider);
    this.session = session;
    root = new FgwPath(this, "/", Collections.EMPTY_LIST);
}

// s3文件系统
public S3FileSystem(FileSystemProvider fileSystemProvider, SessionContext session) {
    super(fileSystemProvider);
    this.session = session;
}
```
- FileSystemProvider(直接调用FileSystem绑定的FileSystemProvider接口))
```
// public abstract class FileSystemProvider.class
public abstract String getScheme();
public abstract FileSystem newFileSystem(URI uri, Map<String,?> env)throws IOException;
public abstract FileSystem getFileSystem(URI uri);
public abstract Path getPath(URI uri);
public FileSystem newFileSystem(Path path, Map<String,?> env)throws IOException{throw new UnsupportedOperationException();}
public FileChannel newFileChannel(Path path, Set<? extends OpenOption> options, FileAttribute<?>... attrs) throws IOException{throw new UnsupportedOperationException(); }
public AsynchronousFileChannel newAsynchronousFileChannel(Path path,Set<? extends OpenOption> options,ExecutorService executor, FileAttribute<?>... attrs)
throws IOException {throw new UnsupportedOperationException();}
public abstract SeekableByteChannel newByteChannel(Path path,Set<? extends OpenOption> options, FileAttribute<?>... attrs) throws IOException;
public abstract DirectoryStream<Path> newDirectoryStream(Path dir, DirectoryStream.Filter<? super Path> filter) throws IOException;
public abstract void createDirectory(Path dir, FileAttribute<?>... attrs) throws IOException;
public void createSymbolicLink(Path link, Path target, FileAttribute<?>... attrs)throws IOException { throw new UnsupportedOperationException(); }
public void createLink(Path link, Path existing) throws IOException { throw new UnsupportedOperationException();}
public abstract void delete(Path path) throws IOException;
public boolean deleteIfExists(Path path) throws IOException 
public Path readSymbolicLink(Path link) throws IOException { throw new UnsupportedOperationException();}
public abstract void copy(Path source, Path target, CopyOption... options) throws IOException;
public abstract void move(Path source, Path target, CopyOption... options) throws IOException;
public abstract boolean isSameFile(Path path, Path path2) throws IOException;
public abstract boolean isHidden(Path path) throws IOException;
public abstract FileStore getFileStore(Path path) throws IOException;
public abstract void checkAccess(Path path, AccessMode... modes) throws IOException;
public abstract <V extends FileAttributeView> V getFileAttributeView(Path path, Class<V> type, LinkOption... options);
public abstract <A extends BasicFileAttributes> A readAttributes(Path path, Class<A> type, LinkOption... options) throws IOException;
public abstract Map<String,Object> readAttributes(Path path, String attributes, LinkOption... options)throws IOException;
public abstract void setAttribute(Path path, String attribute,Object value, LinkOption... options)throws IOException;
```

读取属性：provider#readAttributes 每次操作都会读取文件或目录属性
创建channel :#newFileChannel 操作文件才会创建
读文件：provider#newFileChannel - read 分块读取 FileHandle.read接收到-1时结束读取
写文件：provider#newFileChannel - write 分块写入 #provider#move 拖拉上传【=move】写完后会调用move方法 直接创建【=write】
删除文件：provider#delete
创建文件夹：
provider#createDirectory 先查询查询属性provider#readAttributes，
AbstractSftpSubsystemHelper#doMakeDirectory接收到NoSuchFileException 才会调用provider#createDirectory创建
删除文件夹：provider#delete 先把文件夹内文件一个个删除，再删除文件夹
复制文件夹：先创建文件夹，再把一个个文件写入进文件夹
provider#createDirectory 创建文件夹
provider#newFileChannel - write 写文件
重命名：provider#move 移动
覆盖：provider#writer 写文件(后缀.partfile路径) provider#delete 删除(原文件路径) provider#move移动(后缀.partfile路径)
移动：provider#move 文件和文件夹都只会调用一次move方法