```java
@Service
public class DashboardManagementService {

    private final DashboardService dashboardService;

    @Autowired
    public DashboardManagementService(DashboardService dashboardService) {
        this.dashboardService = dashboardService;
    }

    @Cacheable(
        cacheNames = "dashboardCache",
        key = "'findAll_page_' + #pageable.pageNumber + '_' + #pageable.pageSize + '_' + #root.methodName + '_' + #pageable.sort.toString()"
    )
    public Page<DashboardDTO> findAll(Pageable pageable) {
        return dashboardService.findAll(pageable);
    }

    @Cacheable(cacheNames = "dashboardCache", key = "'get_' + #id")
    public DashboardDTO get(final Long id) {
        return dashboardService.get(id);
    }

    @CacheEvict(cacheNames = "dashboardCache", allEntries = true)
    public DashboardDTO create(final DashboardDetails dashboardDetails) {
        return dashboardService.create(dashboardDetails);
    }

    @Caching(evict = {
        @CacheEvict(cacheNames = "dashboardCache", key = "'get_' + #id"),   // evict single item
        @CacheEvict(cacheNames = "dashboardCache", allEntries = true)      // evict pagination caches
    })
    public DashboardDTO update(final Long id, final DashboardUpdate dashboardUpdate) {
        return dashboardService.update(id, dashboardUpdate);
    }

    @Caching(evict = {
        @CacheEvict(cacheNames = "dashboardCache", key = "'get_' + #id"),   // evict single item
        @CacheEvict(cacheNames = "dashboardCache", allEntries = true)      // evict pagination caches
    })
    public void delete(final Long id) {
        dashboardService.delete(id);
    }
}

@EnableCaching
@SpringBootTest
@ContextConfiguration(classes = {DashboardManagementService.class, CacheConfig.class})
class DashboardManagementServiceTest {

    @MockBean private DashboardService dashboardService;

    @Autowired private DashboardManagementService managementService;

    @Autowired private CacheManager cacheManager;

    @BeforeEach
    void setUp() {
        // Clear the cache before each test
        cacheManager.getCache("dashboardCache").clear();
    }

    @Test
    void testFindAll_isCached() {
        Pageable pageable = PageRequest.of(0, 5);
        Page<DashboardDTO> expected = new PageImpl<>(List.of(new DashboardDTO()));

        when(dashboardService.findAll(pageable)).thenReturn(expected);

        var firstCall = managementService.findAll(pageable);
        var secondCall = managementService.findAll(pageable);

        assertSame(firstCall, secondCall);
        verify(dashboardService, times(1)).findAll(pageable);
    }

    @Test
    void testGet_isCached() {
        Long id = 1L;
        var dto = new DashboardDTO();
        dto.setId(id);
        dto.setName("Test Dashboard");

        when(dashboardService.get(id)).thenReturn(dto);

        var firstCall = managementService.get(id);
        var secondCall = managementService.get(id);

        assertSame(firstCall, secondCall);
        verify(dashboardService, times(1)).get(id);
    }

    @Test
    void testCreate_cacheEvicts() {
        Long id = 1L;
        var dto = new DashboardDTO();
        dto.setId(id);

        when(dashboardService.get(id)).thenReturn(dto);

        // cache it
        var firstCall = managementService.get(id);
        assertEquals(dto, firstCall);

        // create triggers eviction
        managementService.create(new DashboardDetails("Dash", "Desc", "user"));

        // fetch again → should trigger service call again
        managementService.get(id);

        verify(dashboardService, times(2)).get(id);
    }

    @Test
    void testUpdate_cacheEvicts() {
        Long id = 1L;
        var dto = new DashboardDTO();
        dto.setId(id);

        when(dashboardService.get(id)).thenReturn(dto);

        // cache it
        managementService.get(id);

        // update triggers eviction
        managementService.update(id, new DashboardUpdate("New Name", "New Desc"));

        // fetch again → service called again
        managementService.get(id);

        verify(dashboardService, times(2)).get(id);
    }

    @Test
    void testDelete_cacheEvicts() {
        Long id = 1L;
        var dto = new DashboardDTO();
        dto.setId(id);

        when(dashboardService.get(id)).thenReturn(dto);

        // cache it
        managementService.get(id);

        // delete triggers eviction
        managementService.delete(id);

        // fetch again → service called again
        managementService.get(id);

        verify(dashboardService, times(2)).get(id);
    }
}
```
