```java
@SpringBootTest
@ActiveProfiles("test")
@Transactional
class ApplicationServiceTest {

    @Autowired
    private ApplicationService applicationService;

    @Autowired
    private ApplicationRepository applicationRepository;

    @Autowired
    private ApplicationMapper applicationMapper;

    @Test
    void findAll() {
        Application app1 = new Application();
        app1.setName("App1");

        Application app2 = new Application();
        app2.setName("App2");

        applicationRepository.saveAll(List.of(app1, app2));

        List<ApplicationDTO> results = applicationService.findAll();
        assertEquals(2, results.size());
    }

    @Test
    void get() {
        Application app = new Application();
        app.setName("GetApp");
        Application saved = applicationRepository.save(app);

        ApplicationDTO dto = applicationService.get(saved.getId());

        assertNotNull(dto);
        assertEquals(saved.getId(), dto.getId());
        assertEquals("GetApp", dto.getName());
    }

    @Test
    void findByID() {
        Application app = new Application();
        app.setName("FindApp");
        Application saved = applicationRepository.save(app);

        Application found = applicationService.findByID(saved.getId());

        assertNotNull(found);
        assertEquals(saved.getId(), found.getId());
        assertEquals("FindApp", found.getName());
    }

    @Test
    void create() {
        ApplicationDTO dto = new ApplicationDTO();
        dto.setName("CreateApp");

        Long id = applicationService.create(dto);

        assertNotNull(id);
        Application saved = applicationRepository.findById(id).orElseThrow();
        assertEquals("CreateApp", saved.getName());
    }
}


```
