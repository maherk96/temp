```java

package com.example.qaportal.heatmap;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.*;
import org.springframework.data.jpa.domain.Specification;

import jakarta.persistence.criteria.*;
import java.util.List;

import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

class HeatmapSpecificationTest {

    @Mock private Root<Heatmap> root;
    @Mock private CriteriaQuery<?> query;
    @Mock private CriteriaBuilder cb;

    @Mock private Path<Object> namePath;
    @Mock private Path<Object> descPath;
    @Mock private Path<Object> userPath;
    @Mock private Path<Object> userIdPath;
    @Mock private Path<Object> isDefaultPath;
    @Mock private Path<Object> isDeletedPath;

    @Mock private Predicate pSearch1;
    @Mock private Predicate pSearch2;
    @Mock private Predicate pSearchCombined;

    @Mock private Predicate pMyHeatmap;
    @Mock private Predicate pCreatedOthers1;
    @Mock private Predicate pCreatedOthers2;
    @Mock private Predicate pCreatedCombined;

    @Mock private Predicate pSystemHeatmap;
    @Mock private Predicate pDeletedFalse;

    @Mock private Predicate pFinal;

    @BeforeEach
    void setup() {
        MockitoAnnotations.openMocks(this);

        // General mocks
        when(root.get("name")).thenReturn(namePath);
        when(root.get("description")).thenReturn(descPath);
        when(root.get("user")).thenReturn(userPath);
        when(userPath.get("id")).thenReturn(userIdPath);
        when(root.get("isDefault")).thenReturn(isDefaultPath);
        when(root.get("deleted")).thenReturn(isDeletedPath);
    }

    // -------------------------------------------------------------------------

    @Test
    void test_searchPredicateAdded_whenSearchProvided() {
        when(cb.lower(namePath)).thenReturn(namePath);
        when(cb.lower(descPath)).thenReturn(descPath);

        when(cb.like(namePath, "%abc%")).thenReturn(pSearch1);
        when(cb.like(descPath, "%abc%")).thenReturn(pSearch2);
        when(cb.or(pSearch1, pSearch2)).thenReturn(pSearchCombined);

        Specification<Heatmap> spec =
                HeatmapSpecification.build("abc", null, 123L, true);

        Predicate returned = spec.toPredicate(root, query, cb);

        assertNotNull(returned);
        verify(cb).like(namePath, "%abc%");
        verify(cb).like(descPath, "%abc%");
    }

    // -------------------------------------------------------------------------

    @Test
    void test_noSearchPredicate_whenSearchBlank() {
        Specification<Heatmap> spec =
                HeatmapSpecification.build("   ", null, 123L, true);

        Predicate returned = spec.toPredicate(root, query, cb);
        assertNotNull(returned);

        verify(cb, never()).like(any(), anyString());
    }

    // -------------------------------------------------------------------------

    @Test
    void test_myHeatmap_predicateAdded() {
        when(cb.equal(userIdPath, 99L)).thenReturn(pMyHeatmap);

        Specification<Heatmap> spec =
                HeatmapSpecification.build(null, List.of(HeatmapType.MY_HEATMAP), 99L, true);

        Predicate returned = spec.toPredicate(root, query, cb);

        assertNotNull(returned);
        verify(cb).equal(userIdPath, 99L);
    }

    // -------------------------------------------------------------------------

    @Test
    void test_createdByOthers_predicateAdded() {
        when(cb.notEqual(userIdPath, 50L)).thenReturn(pCreatedOthers1);
        when(cb.isFalse(isDefaultPath)).thenReturn(pCreatedOthers2);
        when(cb.and(pCreatedOthers1, pCreatedOthers2)).thenReturn(pCreatedCombined);

        Specification<Heatmap> spec =
                HeatmapSpecification.build(null, List.of(HeatmapType.CREATED_BY_OTHERS), 50L, true);

        Predicate returned = spec.toPredicate(root, query, cb);

        assertNotNull(returned);
        verify(cb).notEqual(userIdPath, 50L);
        verify(cb).isFalse(isDefaultPath);
    }

    // -------------------------------------------------------------------------

    @Test
    void test_systemHeatmap_predicateAdded() {
        when(cb.isTrue(isDefaultPath)).thenReturn(pSystemHeatmap);

        Specification<Heatmap> spec =
                HeatmapSpecification.build(null, List.of(HeatmapType.SYSTEM_HEATMAP), 99L, true);

        Predicate returned = spec.toPredicate(root, query, cb);
        assertNotNull(returned);

        verify(cb).isTrue(isDefaultPath);
    }

    // -------------------------------------------------------------------------

    @Test
    void test_multipleHeatmapTypes_orCombined() {
        when(cb.equal(userIdPath, 1L)).thenReturn(pMyHeatmap);
        when(cb.isTrue(isDefaultPath)).thenReturn(pSystemHeatmap);

        Predicate[] orArray = {pMyHeatmap, pSystemHeatmap};
        when(cb.or(orArray)).thenReturn(pCreatedCombined);

        Specification<Heatmap> spec =
                HeatmapSpecification.build(null,
                        List.of(HeatmapType.MY_HEATMAP, HeatmapType.SYSTEM_HEATMAP),
                        1L,
                        true);

        Predicate result = spec.toPredicate(root, query, cb);

        assertNotNull(result);
        verify(cb).or(orArray);
    }

    // -------------------------------------------------------------------------

    @Test
    void test_deletedFilter_isAddedWhenNotIncluded() {
        when(cb.isFalse(isDeletedPath)).thenReturn(pDeletedFalse);

        Specification<Heatmap> spec =
                HeatmapSpecification.build(null, null, 10L, false);

        Predicate returned = spec.toPredicate(root, query, cb);

        assertNotNull(returned);
        verify(cb).isFalse(isDeletedPath);
    }

    // -------------------------------------------------------------------------

    @Test
    void test_deletedFilter_notAddedWhenIncluded() {
        Specification<Heatmap> spec =
                HeatmapSpecification.build(null, null, 10L, true);

        Predicate returned = spec.toPredicate(root, query, cb);

        assertNotNull(returned);
        verify(cb, never()).isFalse(isDeletedPath);
    }

    // -------------------------------------------------------------------------

    @Test
    void test_fullCombination_ofPredicates() {

        // mock search
        when(cb.lower(namePath)).thenReturn(namePath);
        when(cb.lower(descPath)).thenReturn(descPath);
        when(cb.like(namePath, "%heat%")).thenReturn(pSearch1);
        when(cb.like(descPath, "%heat%")).thenReturn(pSearch2);
        when(cb.or(pSearch1, pSearch2)).thenReturn(pSearchCombined);

        // mock type group (MY_HEATMAP)
        when(cb.equal(userIdPath, 77L)).thenReturn(pMyHeatmap);

        // deleted = false
        when(cb.isFalse(isDeletedPath)).thenReturn(pDeletedFalse);

        // final AND
        Predicate[] finalArray = {pSearchCombined, pMyHeatmap, pDeletedFalse};
        when(cb.and(finalArray)).thenReturn(pFinal);

        Specification<Heatmap> spec =
                HeatmapSpecification.build(
                        "heat",
                        List.of(HeatmapType.MY_HEATMAP),
                        77L,
                        false
                );

        Predicate result = spec.toPredicate(root, query, cb);

        assertEquals(pFinal, result);
        verify(cb).and(finalArray);
    }
}

```
