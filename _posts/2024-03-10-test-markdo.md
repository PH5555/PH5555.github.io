---
layout: post
title: Spring - 파일 업로드
tags: [Spring, 스프링 MVC]
comments: true
---

## 파일 업로드의 폼 차이

HTML 폼 전송 방식에는 두가지가 있다

- application/x-www-form-urlencoded

  일반적으로 데이터를 서버로 전송하는 방식이다. 아무런 설정이 없으면 이 값이 기본값이 된다.
  `data=4&age=3` 과 같이 데이터를 `&`로 구분한다.
  
- multipart/form-data

  파일은 문자가 아니라 바이너리 데이터이기 때문에 문자로 전송하는 방식으로는 전송하지 못한다.
  게다가 파일을 전송할 때에는 문자도 같이 전송될 수도 있다.

![form](/assets/img/formType.PNG)

`multipart/form-data` 형식은 파일형식과 함께 다양한 형식의 데이터들을 보낼 수 있다. Http메시지를 보면 각각의 항목들이 구분되어있다.(---------xxx) 항목별로 헤더가 있고 부가정보가 있다. 이렇게 각각의 항목을 구분해서 한번에 전송해준다.

## 파일 업로드 구현

```
@Controller
@RequestMapping("/servlet/v1")
public class ServletUploadControllerV1 {
    @GetMapping("upload")
    public String upload() {
        return "upload-form";
    }

    @PostMapping("upload")
    public String saveFile(HttpServletRequest request) throws ServletException, IOException {
        log.info("request={}", request);

        String itemName = request.getParameter("itemName");
        log.info("itemName={}", itemName);

        Collection<Part> parts = request.getParts();
        log.info("parts={}", parts);
        return "upload-form";
    }
}
```

- `request.getParts()`를 이용해서 항목들을 가져올 수 있다.

> spring.servlet.multipart.max-file-size=1MB : 파일의 최대 사이즈를 정할 수 있다.

> spring.servlet.multipart.max-request-size=10MB : 파일들의 전체 최대 사이즈를 정할 수 있다.

```java
@Controller
@RequestMapping("/servlet/v2")
public class ServletUploadControllerV2 {
    @Value("${file.dir}")
    private String fileDir;

    @GetMapping("upload")
    public String upload() {
        return "upload-form";
    }

    @PostMapping("upload")
    public String saveFile(HttpServletRequest request) throws ServletException, IOException {
        log.info("request={}", request);

        String itemName = request.getParameter("itemName");
        log.info("itemName={}", itemName);

        Collection<Part> parts = request.getParts();
        log.info("parts={}", parts);

        for (Part part : parts) {
            log.info("______________________");
            String name = part.getName();
            log.info("name={}", name);
            Collection<String> headerNames = part.getHeaderNames();
            for (String headerName : headerNames) {
                String header = part.getHeader(headerName);
                log.info("header name={} , header={}", headerName, header);
            }

            log.info("file nme = {}", part.getSubmittedFileName());

            InputStream inputStream = part.getInputStream();
            String body = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);
            log.info("body={}", body);

            if(StringUtils.hasText(part.getSubmittedFileName())){
                String fullPath = fileDir + part.getSubmittedFileName();
                log.info(fullPath);
                part.write(fullPath);
            }
        }
        return "upload-form";
    }
}
```

- `part.getSubmittedFileName()`를 이용해서 파일의 이름을 알 수 있다.
- `part.getInputStream()`를 이용해서 파일의 내용을 읽을 수 있다.
-  `part.write(fullPath)`를 이용해서 fullPath 의 경로에 파일을 저장할 수 있다.

# 스프링의 파일 업로드

```java
@Controller
@RequestMapping("/spring")
public class SpringUploadController {
    @Value("${file.dir}")
    private String fileDir;

    @GetMapping("upload")
    public String upload() {
        return "upload-form";
    }

    @PostMapping("upload")
    public String saveFile(@RequestParam String itemName, @RequestParam MultipartFile file, HttpServletRequest request) throws ServletException, IOException {
        if(!file.isEmpty()){
            String fullPath = fileDir + file.getOriginalFilename();
            log.info(fullPath);
            file.transferTo(new File(fullPath));
        }
        return "upload-form";
    }
}
```

- 폼에서 `MultipartFile` 타입 @RequestParam으로 파일을 가져올 수 있다. @ModelAttribute에서도 MultipartFile로 가져올 수 있다.
- getOriginalFilename()으로 파일의 이름을 가져올 수 있다.
- transferTo() 으로 지정한 경로로 파일을 저장할 수 있다.

# 파일 업로드 예제

```java
@Data
public class Item {
    private Long id;
    private String itemName;
    private UploadFile attachFile;
    private List<UploadFile> imageFiles;
}
```

```java
@Repository
public class ItemRepository {
    private final Map<Long, Item> store = new HashMap<>();
    private Long sequence = 0L;

    public Item save(Item item) {
        item.setId(++sequence);
        store.put(item.getId(), item);
        return item;
    }

    public Item findById(Long id) {
        return store.get(id);
    }
}
```

```java
@Data
public class UploadFile {
    private String uploadFileName;
    private String storeFileName;

    public UploadFile(String uploadFileName, String storeFileName) {
        this.uploadFileName = uploadFileName;
        this.storeFileName = storeFileName;
    }
}
```

도메인들과 레파지토리를 만들어준다.

> 사용자들이 다른 내용인데 이름이 똑같은 파일을 올릴 수도 있다. 저장소에 같은 이름으로 저장하면 덮어씌워지므로, uuid로 storeFileName을 만들어서 안 겹치게 한다.

```java
@Component
public class FileStore {
    @Value("${file.dir}")
    private String fileDir;

    public String getFullPath(String fileName){
        return fileDir + fileName;
    }

    public List<UploadFile> storeFiles(List<MultipartFile> files) throws IOException {
        List<UploadFile> storeFileResult = new ArrayList<>();
        for (MultipartFile file : files) {
            if (!file.isEmpty()) {
                storeFileResult.add(storeFile(file));
            }
        }
        return storeFileResult;
    }

    public UploadFile storeFile(MultipartFile file) throws IOException {
        if(file.isEmpty()) {
           return null;
        }

        String originalFilename = file.getOriginalFilename();

        String storeFileName = createStoreFileName(originalFilename);
        log.info("store name={}",storeFileName);
        file.transferTo(new File(getFullPath(storeFileName)));

        return new UploadFile(originalFilename, storeFileName);
    }

    private String createStoreFileName(String originalFilename) {
        String uuid = UUID.randomUUID().toString();
        String ext = extractExt(originalFilename);

        return uuid + "." + ext;
    }

    private String extractExt(String originalFilename) {
        int index = originalFilename.lastIndexOf(".");
        return originalFilename.substring(index + 1);
    }
}
```

- `@Value`를 이용해서 application.properties에 있는 값을 가져올 수 있다.
- 파일을 저장하는 로직을 만든다. 파일마다 이름이 안 겹치게 하도록 UUID를 이용해서 파일명을 만들고 확장자는 기존 파일의 확장자를 가져온다. 파일을 여러개 저장하는 메서드도 파일 하나를 저장하는 로직을 사용해서 `UploadFile`의 리스트를 반환하도록 만든다.

```java
@Data
public class ItemForm {
    private Long id;
    private String itemName;
    private MultipartFile attachFile;
    private List<MultipartFile> imageFiles;
}
```

상품 저장용 폼을 만들어준다. Item은 데이터베이스에 저장되어야하는 데이터이므로 UploadFile 형을 이용한다.(문자열) 데이터 베이스에 저장할 때는 실제 파일을 저장하는 것이 아니라 파일 이름을 저장한다.

```java
@Controller
@Slf4j
@RequiredArgsConstructor
public class ItemController {
    private final ItemRepository itemRepository;
    private final FileStore fileStore;

    @GetMapping("/items/new")
    public String newItem(@ModelAttribute ItemForm itemForm){
        return "item-form";
    }

    @PostMapping("items/new")
    public String saveItems(@ModelAttribute ItemForm itemForm, RedirectAttributes redirectAttributes) throws IOException {
        UploadFile attachFile = fileStore.storeFile(itemForm.getAttachFile());
        List<UploadFile> imageFiles = fileStore.storeFiles(itemForm.getImageFiles());

        Item item = new Item();
        item.setItemName(itemForm.getItemName());
        item.setAttachFile(attachFile);
        item.setImageFiles(imageFiles);

        itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", item.getId());
        return "redirect:/items/{itemId}";
    }

    @GetMapping("items/{itemId}")
    public String itemView(@PathVariable Long itemId, Model model) {
        Item item = itemRepository.findById(itemId);
        model.addAttribute("item", item);
        return "item-view";
    }

    @ResponseBody
    @GetMapping("/images/{filename}")
    public Resource downloadImage(@PathVariable String filename) throws MalformedURLException {
        return new UrlResource("file:" + fileStore.getFullPath(filename));
    }

    @GetMapping("/attach/{itemId}")
    public ResponseEntity<Resource> downloadAttach(@PathVariable Long itemId) throws MalformedURLException {
        Item item = itemRepository.findById(itemId);
        UploadFile attachFile = item.getAttachFile();
        String storeFileName = attachFile.getStoreFileName();
        String uploadFileName = attachFile.getUploadFileName();

        UrlResource resource = new UrlResource("file:" + fileStore.getFullPath(storeFileName));
        String encodedFileName = UriUtils.encode(uploadFileName, StandardCharsets.UTF_8);
        String contentDisposition = "attachment; filename=\"" + encodedFileName + "\"";

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, contentDisposition)
                .body(resource);
    }
}
```

상품의 정보를 저장하고 보여주는 컨트롤러를 개발한다.

- `downloadImage` 메서드는 이미지를 보여주는 기능이다. html에서 img의 src에서 해당 컨트롤러에 사진을 요청한다. 파일의 path를 이용해서 UrlResource를 반환해준다. 모든 사용자가 다 해당 사진에 접근할 수 있기 때문에 보안에는 취약하다.
- `downloadAttach` 메서드는 파일을 다운할 수 있게 해주는 기능이다. 이미지를 보여줄때와 마찬가지로 UrlResouce를 만들어서 body에 넣어주면 된다. 하지만 그냥 body에만 넣어주면 해당 파일의 내용을 보여줄 뿐이다. header에 이 요청은 다운로드를 위한 요청이라는것을 명시해주어야한다. 그 부분이 `String contentDisposition = "attachment; filename=\"" + encodedFileName + "\"";` 이다.
