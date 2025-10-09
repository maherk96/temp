```java
public Page<DashboardDTO> findAll(String search, String dashboardType, String currentUser, Pageable pageable) {
    Page<Dashboard> dashboards;

    // Search filter (optional)
    if (StringUtils.hasText(search)) {
        dashboards = dashboardRepository.findByNameContainingIgnoreCaseOrDescriptionContainingIgnoreCase(
                search, search, pageable);
    } else {
        dashboards = dashboardRepository.findAll(pageable);
    }

    // Map entities to DTOs
    List<DashboardDTO> dashboardDTOs = dashboards
            .map(dashboard -> mapToDTO(dashboard, new DashboardDTO()))
            .getContent();

    // Apply dashboardType filter if provided
    if (StringUtils.hasText(dashboardType)) {
        dashboardDTOs = dashboardDTOs.stream()
                .filter(dto -> {
                    if ("MY_DASHBOARDS".equalsIgnoreCase(dashboardType)) {
                        return currentUser.equals(String.valueOf(dto.getUser()));
                    } else if ("CREATED_BY_OTHERS".equalsIgnoreCase(dashboardType)) {
                        return dto.getUser() != null && !currentUser.equals(String.valueOf(dto.getUser()));
                    } else if ("SYSTEM_DASHBOARDS".equalsIgnoreCase(dashboardType)) {
                        return dto.getUser() == null;
                    }
                    return true; // unknown type, skip filtering
                })
                .toList();
    }

    return new PageImpl<>(dashboardDTOs, pageable, dashboards.getTotalElements());
}


 Page<Dashboard> findByNameContainingIgnoreCaseOrDescriptionContainingIgnoreCase(
            String name, String description, Pageable pageable);
```
