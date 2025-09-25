```java


public class FieldErrorDetails {
    private String field;
    private String code;      // validation error code (e.g. NotBlank)
    private String message;   // human-friendly message (e.g. "Name must not be blank")

    public FieldErrorDetails() {}

    public FieldErrorDetails(String field, String code, String message) {
        this.field = field;
        this.code = code;
        this.message = message;
    }

    // Getters and setters
}

public class ErrorResponse {

    private Instant timestamp;
    private int status;              // HTTP status code (e.g. 404)
    private String error;            // Human-readable status (e.g. "Not Found")
    private String message;          // Exception message
    private String path;             // Request path (/api/dashboard/1)
    private List<FieldErrorDetails> fieldErrors;

    public ErrorResponse() {}

    public ErrorResponse(Instant timestamp, int status, String error, String message, String path, List<FieldErrorDetails> fieldErrors) {
        this.timestamp = timestamp;
        this.status = status;
        this.error = error;
        this.message = message;
        this.path = path;
        this.fieldErrors = fieldErrors;
    }

    // Getters and setters omitted for brevity
}


@RestControllerAdvice(annotations = RestController.class)
public class RestExceptionHandler {

    /**
     * Handle NotFoundException (custom domain exception)
     */
    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(NotFoundException ex, HttpServletRequest request) {
        return buildErrorResponse(ex, HttpStatus.NOT_FOUND, request, null);
    }

    /**
     * Handle validation errors from @Valid
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(MethodArgumentNotValidException ex,
                                                                HttpServletRequest request) {
        List<FieldErrorDetails> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
                .map(err -> new FieldErrorDetails(
                        err.getField(),
                        err.getCode(),
                        err.getDefaultMessage()
                ))
                .toList();

        return buildErrorResponse(ex, HttpStatus.BAD_REQUEST, request, fieldErrors);
    }

    /**
     * Handle Spring's ResponseStatusException (e.g. from WebFlux / custom responses)
     */
    @ExceptionHandler(ResponseStatusException.class)
    public ResponseEntity<ErrorResponse> handleResponseStatus(ResponseStatusException ex,
                                                              HttpServletRequest request) {
        return buildErrorResponse(ex, ex.getStatusCode(), request, null);
    }

    /**
     * Handle custom invalid pagination request
     */
    @ExceptionHandler(InvalidResponseException.class)
    public ResponseEntity<ErrorResponse> handleInvalidPagination(InvalidResponseException ex,
                                                                 HttpServletRequest request) {
        return buildErrorResponse(ex, HttpStatus.BAD_REQUEST, request, ex.getErrorDetails());
    }

    /**
     * Generic fallback for all unhandled exceptions
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex, HttpServletRequest request) {
        return buildErrorResponse(ex, HttpStatus.INTERNAL_SERVER_ERROR, request, null);
    }

    /**
     * Helper method to construct ErrorResponse consistently
     */
    private ResponseEntity<ErrorResponse> buildErrorResponse(Exception ex,
                                                             HttpStatus status,
                                                             HttpServletRequest request,
                                                             List<FieldErrorDetails> fieldErrors) {
        ErrorResponse response = new ErrorResponse(
                Instant.now(),
                status.value(),
                status.getReasonPhrase(),
                ex.getMessage(),
                request.getRequestURI(),
                fieldErrors
        );
        return ResponseEntity.status(status).body(response);
    }
}

@RestController
@RequestMapping(value = "/api/dashboard", produces = MediaType.APPLICATION_JSON_VALUE)
@Tag(name = "QAP Dashboard", description = "QAP Dashboards Service API")
public class QAPDashboardResource {

    private final DashboardManagementService dashboardManagementService;

    public QAPDashboardResource(DashboardManagementService dashboardManagementService) {
        this.dashboardManagementService = dashboardManagementService;
    }

    @PostMapping
    @Operation(summary = "Create a new dashboard", description = "Creates and returns a newly created dashboard.")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "Dashboard created",
            content = @Content(schema = @Schema(implementation = DashboardDTO.class))),
        @ApiResponse(responseCode = "400", description = "Invalid request body",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))),
        @ApiResponse(responseCode = "500", description = "Internal server error",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    public ResponseEntity<DashboardDTO> createDashboard(@Valid @RequestBody DashboardDetails dashboardDetails) {
        DashboardDTO created = dashboardManagementService.create(dashboardDetails);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(created.getId())
                .toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    @Operation(summary = "Update an existing dashboard", description = "Updates a dashboard by its ID.")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Dashboard updated",
            content = @Content(schema = @Schema(implementation = DashboardDTO.class))),
        @ApiResponse(responseCode = "400", description = "Invalid update data",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))),
        @ApiResponse(responseCode = "404", description = "Dashboard not found",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))),
        @ApiResponse(responseCode = "500", description = "Internal server error",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    public ResponseEntity<DashboardDTO> updateDashboard(
            @PathVariable Long id,
            @Valid @RequestBody DashboardUpdate update) {
        DashboardDTO updated = dashboardManagementService.update(id, update);
        return ResponseEntity.ok(updated);
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "Delete an existing dashboard", description = "Deletes a dashboard by its ID.")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "204", description = "Dashboard deleted successfully"),
        @ApiResponse(responseCode = "404", description = "Dashboard not found",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))),
        @ApiResponse(responseCode = "500", description = "Internal server error",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    public ResponseEntity<Void> deleteDashboard(@PathVariable Long id) {
        dashboardManagementService.delete(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    @Operation(summary = "Get paginated dashboards", description = "Returns a paginated list of dashboards.")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "List of dashboards",
            content = @Content(schema = @Schema(implementation = PaginatedResponse.class))),
        @ApiResponse(responseCode = "400", description = "Invalid pagination parameters",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))),
        @ApiResponse(responseCode = "500", description = "Internal server error",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    public ResponseEntity<PaginatedResponse<DashboardDTO>> getDashboards(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "10") int size) {
        Page<DashboardDTO> dashboardPage = dashboardManagementService.findAll(PageRequest.of(page - 1, size));
        return ResponseEntity.ok(PaginatedResponseMapper.from(dashboardPage));
    }
}

@WebMvcTest(QAPDashboardResource.class)
@AutoConfigureMockMvc
class QAPDashboardResourceTest {

    @Autowired private MockMvc mockMvc;

    @MockBean private DashboardManagementService dashboardManagementService;

    private ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void createDashboard_returns201() throws Exception {
        DashboardDTO dto = new DashboardDTO();
        dto.setId(1L);
        dto.setName("My Dashboard");

        when(dashboardManagementService.create(any(DashboardDetails.class)))
                .thenReturn(dto);

        DashboardDetails details = new DashboardDetails("My Dashboard", "Description", "user1");

        mockMvc.perform(post("/api/dashboard")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(details)))
                .andExpect(status().isCreated())
                .andExpect(header().string("Location", containsString("/api/dashboard/1")))
                .andExpect(jsonPath("$.id").value(1L))
                .andExpect(jsonPath("$.name").value("My Dashboard"));
    }

    @Test
    void updateDashboard_returns200() throws Exception {
        DashboardDTO dto = new DashboardDTO();
        dto.setId(1L);
        dto.setName("Updated Name");

        when(dashboardManagementService.update(eq(1L), any(DashboardUpdate.class)))
                .thenReturn(dto);

        DashboardUpdate update = new DashboardUpdate("Updated Name", "New Description");

        mockMvc.perform(put("/api/dashboard/1")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(update)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1L))
                .andExpect(jsonPath("$.name").value("Updated Name"));
    }

    @Test
    void deleteDashboard_returns204() throws Exception {
        mockMvc.perform(delete("/api/dashboard/1"))
                .andExpect(status().isNoContent());

        verify(dashboardManagementService, times(1)).delete(1L);
    }

    @Test
    void getDashboards_returns200() throws Exception {
        DashboardDTO dto = new DashboardDTO();
        dto.setId(1L);
        dto.setName("Test Dash");

        Page<DashboardDTO> page = new PageImpl<>(List.of(dto));
        when(dashboardManagementService.findAll(any(Pageable.class)))
                .thenReturn(page);

        mockMvc.perform(get("/api/dashboard?page=1&size=10"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content[0].id").value(1L))
                .andExpect(jsonPath("$.content[0].name").value("Test Dash"));
    }

    @Test
    void getDashboard_notFound_returns404() throws Exception {
        when(dashboardManagementService.update(eq(99L), any(DashboardUpdate.class)))
                .thenThrow(new NotFoundException("Dashboard with id 99 not found"));

        DashboardUpdate update = new DashboardUpdate("Name", "Desc");

        mockMvc.perform(put("/api/dashboard/99")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(update)))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.status").value(404))
                .andExpect(jsonPath("$.error").value("Not Found"))
                .andExpect(jsonPath("$.message").value("Dashboard with id 99 not found"));
    }
}
```
