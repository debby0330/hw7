# ARIA v4.0 - Hualien Disaster Accessibility Assessment

## Project Overview
Integrated automatic disaster accessibility assessment combining road network,
rainfall dynamics, and terrain risk data for Hualien City, Taiwan.

## Data Sources
- **Road network** : OpenStreetMap via OSMnx (auto-downloaded)
- **Rainfall**     : Week 6 Kriging interpolation (or built-in simulation)
- **Terrain risk** : Week 4 shelter audit → terrain_risk_audit.json
- **Shelters**     : Week 3 shelter locations & river distances

## AI Diagnostic Log

### 1. OSMnx Extraction
**Issue** : [describe any timeout / incomplete network issues]
**Solution** : [e.g. increased ox.settings.timeout = 300]

### 2. Isochrone Polygon Anomaly
**Issue** : [describe shape issues, holes, etc.]
**Solution** : [e.g. switched from concave_hull to convex_hull]

### 3. Terrain Risk Overlay
**Issue** : JSON has no lat/lon → geocoding required
**Solution** : Used geopy Nominatim + sjoin_nearest (max_distance=2 km)

## Key Findings
- Most fragile bottleneck : [fill after running]
- Maximum accessibility loss : [fill after running]
- Rescue priority order : [fill after running]

## Submission Checklist
- [ ] ARIA_v4.ipynb (complete with outputs)
- [ ] output/hualien_network.graphml
- [ ] output/bottleneck_map.png
- [ ] output/isochrone_comparison.png
- [ ] output/accessibility_table.csv
- [ ] output/ai_strategy_report.txt (if Gemini key provided)
- [ ] README.md (this file)
