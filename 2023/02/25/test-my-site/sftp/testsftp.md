- dependcy
```
<dependency>
    <groupId>org.apache.sshd</groupId>
    <artifactId>sshd-common</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.sshd</groupId>
    <artifactId>sshd-core</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.sshd</groupId>
    <artifactId>sshd-netty</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.sshd</groupId>
    <artifactId>sshd-sftp</artifactId>
</dependency>
```

- interface
```
@RestController
@RequestMapping("/sftp")
public class SftpManagementController {
    private static final String MODULE = "serviceManage";
    @Autowired
    private SftpServerServiceImpl sftpServerServiceImpl;
    @RequestMapping(value = "/start", method = RequestMethod.POST)
    public BaseResponse startService(final HttpServletRequest request) {
        LOGGER.info(String.format(Locale.ROOT, "module=%s, action=sftp_service_start", MODULE));
        BaseResponse resp = new BaseResponse();
        try {
            sftpServerServiceImpl.start();
        } catch (IOException | FgwException ex) {
            LOGGER.error(ExceptionUtils.getStackTrace(ex));
            return BaseResponse.ok().setCode(HttpStatus.INTERNAL_SERVER_ERROR.value()).setMessage(ex.getMessage());
        }
        return BaseResponse.ok().setMessage("success");
    }
    @RequestMapping(value = "/stop", method = RequestMethod.POST)
    public BaseResponse stopService(final HttpServletRequest request) {
        LOGGER.info(String.format(Locale.ROOT, "module=%s, action=sftp_service_stop", MODULE));
        String message;
        try {
            message = sftpServerServiceImpl.stop();
        } catch (IOException ex) {
            LOGGER.error(ExceptionUtils.getStackTrace(ex));
            return BaseResponse.ok().setCode(HttpStatus.INTERNAL_SERVER_ERROR.value()).setMessage(ex.getMessage());
        }
        return BaseResponse.ok().setMessage(message);
    }
    @RequestMapping(value = "/status", method = RequestMethod.POST)
    public BaseResponse checkService(final HttpServletRequest request) {
        LOGGER.info(String.format(Locale.ROOT, "module=%s, action=sftp_service_check", MODULE));
        Map<String, Object> check = sftpServerServiceImpl.getStatus();
        return BaseResponse.ok().setMessage("success").setResult(check);
    }
```

- service
```
@Service
public class SftpServerServiceImpl implements FileServerService {
    private static final Logger LOGGER = LoggerFactory.getLogger(SftpServerServiceImpl.class);
    private static final int IDLE_TIMEOUT = 10;
    private SshServer server;
    @Value("${ftp.sftp.port:22}")
    private int port;
    @Value("${ftp.sftp.host:127.0.0.1}")
    private String host;
    @Value("${ftp.sftp.hostKey.path:}")
    private String hostKeyPath;
    @Value("${ftp.sftp.hostKey.autoGenerator:false}")
    private boolean autoGenerator;
    @Autowired
    private SftpSessionListener sftpSessionListener;
    @Autowired
    private SftpPasswordAuthenticator passwordAuthenticator;
    @Autowired
    private SftpPublicKeyAuthenticator sftpPublicKeyAuthenticator;
    @Autowired
    private FgwFileSystemFactory fgwFileSystemFactory;
    @Autowired
    private SftpEventHandler sftpEventHandler;

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
        server.setFileSystemFactory(fgwFileSystemFactory);
        server.addSessionListener(sftpSessionListener);
        SftpSubsystemFactory factory = new SftpSubsystemFactory();
        factory.addSftpEventListener(sftpEventHandler);
        server.setSubsystemFactories(Collections.singletonList(factory));
        server.setPasswordAuthenticator(passwordAuthenticator);
        server.setPublickeyAuthenticator(sftpPublicKeyAuthenticator);
        server.start();
        LOGGER.info("sftp server started bind {} on {}",
            NormalizerUtil.replaceCRLF(host),
            NormalizerUtil.replaceCRLF(String.valueOf(port))
        );
    }
    private Function<DHFactory, KeyExchangeFactory> dh2kex() {
        return factory -> factory == null ? null : factory.isGroupExchange() ? DHGEXServer.newFactory(factory) : DHGServer.newFactory(factory);
    }
    @Override
    public String stop() throws IOException {
        if (server != null && server.isStarted()) {
            server.stop();
            server = null;
            return "success";
        }
        return "service already stop";
    }
    @Override
    public Map<String, Object> getStatus() {
        Map<String, Object> map = new HashMap<>();
        if (server == null) {
            map.put("active", false);
            map.put("activeCount", 0);
            return map;
        }
        map.put("active", server.isStarted());
        int sessions = 0;
        if (server.isStarted()) {
            sessions = server.getActiveSessions().size();
        }
        map.put("activeCount", sessions);
        return map;
    }
    public SshServer getServer() {return server;}
    public int getPort() {return port;}
    public String getHost() {return host;}
```

- filesystemfactory
```java
@Override
public FileSystem createFileSystem(SessionContext session) throws IOException {
    return getFileSystemProvider(session);
}
private FileSystem getFileSystemProvider(SessionContext session) {
    FileSystemProvider fileSystemProvider;
    if (this.s3StoreEnable) {
        fileSystemProvider = new AiyFileSystemProvider(session, fileSystemService, this.hisS3Config);
        return fileSystemProvider.getFileSystem(URI.create("sftps3:///"));
    }
    fileSystemProvider = new AiyFileSystemProvider(session, fileSystemService);
    return fileSystemProvider.getFileSystem(URI.create("file:///"));
}

//filesystemprovider
public class AiyFileSystemProvider extends FileSystemProvider {
    protected FileSystemProvider proxy = FileSystems.getDefault().provider();
    private final SessionContext session;
    private final FileSystemService fileSystemService;
    private FileSystem fileSystem;
    public AiyFileSystemProvider(SessionContext session, FileSystemService service) {
        this.session = session;
        this.fileSystem = new AiyFileSystem(this, session);
        this.fileSystemService = service;
    }
    public AiyFileSystemProvider(SessionContext session, FileSystemService service, S3Config s3Config) {
        this(session, service);
        if (hisS3Config != null){
            this.proxy = new S3FileSystemProvider(session, service, hisS3Config);
            this.fileSystem = new S3FileSystem(this, session);
        }
    }
}

// filesystem
public class AiyFileSystem extends BaseFileSystem<FgwPath> {}

// path
public class AiyPathextends BasePath<AiyPath, AiyFileSystem> {}
``