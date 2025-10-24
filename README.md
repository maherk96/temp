```java

import org.springframework.jdbc.core.RowMapper;
import java.sql.ResultSet;
import java.sql.SQLException;

public class DashboardWidgetDataMapper implements RowMapper<DashboardWidgetData> {

    @Override
    public DashboardWidgetData mapRow(ResultSet rs, int rowNum) throws SQLException {
        DashboardWidgetData data = new DashboardWidgetData();
        data.setId(rs.getLong("ID"));
        data.setWidgetDescription(rs.getString("WIDGET_DESCRIPTION"));
        data.setWidgetName(rs.getString("WIDGET_NAME"));
        data.setOrdinal(rs.getInt("ORDINAL"));
        data.setDashboardId(rs.getLong("DASHBOARD_ID"));
        data.setDashboardWidgetConfigId(rs.getLong("DASHBOARD_WIDGET_CONFIG_ID"));
        data.setWidgetConfiguration(rs.getString("WIDGET_CONFIGURATION"));
        data.setDashboardName(rs.getString("DASHBOARD_NAME"));
        data.setDashboardDescription(rs.getString("DASHBOARD_DESCRIPTION"));
        return data;
    }
}

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class DashboardWidgetData {
    private Long id;
    private String widgetDescription;
    private String widgetName;
    private Integer ordinal;
    private Long dashboardId;
    private Long dashboardWidgetConfigId;
    private String widgetConfiguration;
    private String dashboardName;
    private String dashboardDescription;
}

```
