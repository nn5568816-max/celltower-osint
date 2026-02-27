# celltower-osint

#!/usr/bin/env python3
"""
Cellular Tower Mobile Signal OSINT Tool
Analyze tower locations, signal strength, and network infrastructure
"""

import requests
import json
import csv
import argparse
import sys
from datetime import datetime
from typing import Dict, List, Tuple
import subprocess
import os

class CellTowerOSINT:
    """Main OSINT tool for cellular tower analysis"""
    
    def __init__(self):
        self.tower_data = []
        self.signal_reports = []
        self.timestamp = datetime.now().isoformat()
        
    def query_opencellid(self, lat: float, lon: float, radius: int = 5000) -> List[Dict]:
        """
        Query OpenCellID database for tower information
        Radius in meters (default 5km)
        """
        print(f"[*] Querying OpenCellID for towers near {lat}, {lon} (radius: {radius}m)...")
        
        try:
            # Using the OpenCellID API (free tier)
            url = f"https://opencellid.org/cell/get?lat={lat}&lon={lon}&range={radius}&format=json"
            
            headers = {
                'User-Agent': 'Mozilla/5.0 (Linux; Android 12) AppleWebKit/537.36'
            }
            
            response = requests.get(url, headers=headers, timeout=10)
            response.raise_for_status()
            
            data = response.json()
            
            if 'cells' in data:
                towers = data['cells']
                print(f"[+] Found {len(towers)} towers")
                self.tower_data.extend(towers)
                return towers
            else:
                print("[-] No towers found in response")
                return []
                
        except requests.exceptions.RequestException as e:
            print(f"[-] Error querying OpenCellID: {e}")
            return []
    
    def analyze_signal_strength(self, signal_readings: List[Dict]) -> Dict:
        """
        Analyze signal strength patterns and coverage
        """
        print("[*] Analyzing signal strength patterns...")
        
        if not signal_readings:
            return {}
        
        rsrp_values = [r.get('rsrp', -200) for r in signal_readings if 'rsrp' in r]
        sinr_values = [r.get('sinr', 0) for r in signal_readings if 'sinr' in r]
        
        analysis = {
            'total_readings': len(signal_readings),
            'avg_rsrp': sum(rsrp_values) / len(rsrp_values) if rsrp_values else None,
            'min_rsrp': min(rsrp_values) if rsrp_values else None,
            'max_rsrp': max(rsrp_values) if rsrp_values else None,
            'avg_sinr': sum(sinr_values) / len(sinr_values) if sinr_values else None,
            'coverage_quality': self._classify_coverage(sum(rsrp_values) / len(rsrp_values)) if rsrp_values else 'unknown'
        }
        
        return analysis
    
    def _classify_coverage(self, avg_rsrp: float) -> str:
        """Classify signal coverage quality based on RSRP"""
        if avg_rsrp > -85:
            return "Excellent"
        elif avg_rsrp > -100:
            return "Good"
        elif avg_rsrp > -110:
            return "Fair"
        elif avg_rsrp > -120:
            return "Poor"
        else:
            return "Very Poor"
    
    def map_tower_locations(self, towers: List[Dict]) -> Dict:
        """
        Create spatial analysis of tower distribution
        """
        print("[*] Mapping tower locations...")
        
        if not towers:
            print("[-] No tower data to map")
            return {}
        
        lats = [t.get('lat') for t in towers if t.get('lat')]
        lons = [t.get('lon') for t in towers if t.get('lon')]
        
        if not lats or not lons:
            return {}
        
        mapping = {
            'center_lat': (max(lats) + min(lats)) / 2,
            'center_lon': (max(lons) + min(lons)) / 2,
            'coverage_north': max(lats),
            'coverage_south': min(lats),
            'coverage_east': max(lons),
            'coverage_west': min(lons),
            'estimated_area_sq_km': self._calculate_coverage_area(lats, lons),
            'tower_count': len(towers)
        }
        
        return mapping
    
    def _calculate_coverage_area(self, lats: List[float], lons: List[float]) -> float:
        """Rough calculation of coverage area in km²"""
        lat_range = max(lats) - min(lats)
        lon_range = max(lons) - min(lons)
        # Approximate at equator: 1 degree ≈ 111km
        area = (lat_range * 111) * (lon_range * 111)
        return round(area, 2)
    
    def identify_tower_types(self, towers: List[Dict]) -> Dict:
        """
        Classify towers by type (macro, small cell, femtocell, etc.)
        """
        print("[*] Classifying tower types...")
        
        tower_types = {
            'macrocells': 0,
            'microcells': 0,
            'picocells': 0,
            'femtocells': 0,
            'unknown': 0
        }
        
        for tower in towers:
            # Classification based on available metadata
            range_val = tower.get('range', 0)
            
            if range_val > 35000:
                tower_types['macrocells'] += 1
            elif range_val > 5000:
                tower_types['microcells'] += 1
            elif range_val > 1000:
                tower_types['picocells'] += 1
            elif range_val > 0:
                tower_types['femtocells'] += 1
            else:
                tower_types['unknown'] += 1
        
        return tower_types
    
    def extract_network_info(self, towers: List[Dict]) -> Dict:
        """
        Extract network operator information
        """
        print("[*] Extracting network information...")
        
        operators = {}
        bands = {}
        technologies = {}
        
        for tower in towers:
            # Operator info
            operator = tower.get('operator', 'Unknown')
            operators[operator] = operators.get(operator, 0) + 1
            
            # Technology info (LTE, 5G, 3G, etc.)
            tech = tower.get('technology', 'Unknown')
            technologies[tech] = technologies.get(tech, 0) + 1
            
            # Band info
            band = tower.get('band', 'Unknown')
            bands[band] = bands.get(band, 0) + 1
        
        return {
            'operators': operators,
            'technologies': technologies,
            'frequency_bands': bands
        }
    
    def export_report(self, filename: str = 'tower_osint_report.json'):
        """Export analysis report to JSON"""
        print(f"[*] Exporting report to {filename}...")
        
        report = {
            'timestamp': self.timestamp,
            'tower_data': self.tower_data,
            'signal_analysis': self.signal_reports,
            'export_date': datetime.now().isoformat()
        }
        
        with open(filename, 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"[+] Report exported to {filename}")
    
    def export_csv(self, filename: str = 'towers.csv'):
        """Export tower data to CSV"""
        print(f"[*] Exporting to CSV: {filename}...")
        
        if not self.tower_data:
            print("[-] No tower data to export")
            return
        
        keys = set()
        for tower in self.tower_data:
            keys.update(tower.keys())
        
        with open(filename, 'w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=sorted(keys))
            writer.writeheader()
            writer.writerows(self.tower_data)
        
        print(f"[+] Exported {len(self.tower_data)} towers to {filename}")
    
    def generate_summary(self) -> str:
        """Generate human-readable summary"""
        summary = f"""
╔════════════════════════════════════════════════════════════╗
║         CELLULAR TOWER OSINT ANALYSIS REPORT               ║
╚════════════════════════════════════════════════════════════╝

Timestamp: {self.timestamp}

TOWER STATISTICS:
  • Total Towers Found: {len(self.tower_data)}
  • Tower Types: {self.identify_tower_types(self.tower_data)}

NETWORK INFORMATION:
{self._format_network_info()}

COVERAGE ANALYSIS:
{self._format_coverage_info()}

Report Generated: {datetime.now().isoformat()}
"""
        return summary
    
    def _format_network_info(self) -> str:
        info = self.extract_network_info(self.tower_data)
        output = ""
        
        if info['operators']:
            output += "  Operators:\n"
            for op, count in info['operators'].items():
                output += f"    - {op}: {count} towers\n"
        
        if info['technologies']:
            output += "\n  Technologies:\n"
            for tech, count in info['technologies'].items():
                output += f"    - {tech}: {count} towers\n"
        
        return output
    
    def _format_coverage_info(self) -> str:
        mapping = self.map_tower_locations(self.tower_data)
        if not mapping:
            return "  No coverage data available"
        
        return f"""  Center: {mapping['center_lat']:.4f}, {mapping['center_lon']:.4f}
  Coverage Area: {mapping['estimated_area_sq_km']} km²
  North: {mapping['coverage_north']:.4f}
  South: {mapping['coverage_south']:.4f}
  East: {mapping['coverage_east']:.4f}
  West: {mapping['coverage_west']:.4f}"""


def main():
    parser = argparse.ArgumentParser(
        description='Cellular Tower Mobile Signal OSINT Tool',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog='''
Examples:
  %(prog)s --lat 40.7128 --lon -74.0060 --radius 10000
  %(prog)s --lat 51.5074 --lon -0.1278 --output tower_report.json
        ''')
    
    parser.add_argument('--lat', type=float, required=True, help='Latitude')
    parser.add_argument('--lon', type=float, required=True, help='Longitude')
    parser.add_argument('--radius', type=int, default=5000, help='Search radius in meters (default: 5000)')
    parser.add_argument('--output', type=str, default='tower_osint_report.json', help='Output JSON file')
    parser.add_argument('--csv', type=str, help='Export to CSV file')
    parser.add_argument('--verbose', '-v', action='store_true', help='Verbose output')
    
    args = parser.parse_args()
    
    tool = CellTowerOSINT()
    
    print("""
╔════════════════════════════════════════════════════════════╗
║    CELLULAR TOWER MOBILE SIGNAL OSINT TOOL v1.0           ║
║    Legal Research & Network Analysis Only                  ║
╚════════════════════════════════════════════════════════════╝
    """)
    
    print(f"[*] Starting OSINT analysis for coordinates: {args.lat}, {args.lon}")
    print(f"[*] Search radius: {args.radius} meters")
    print()
    
    # Query towers
    towers = tool.query_opencellid(args.lat, args.lon, args.radius)
    
    if towers:
        # Analysis
        print("\n[*] Running analysis modules...")
        
        tower_types = tool.identify_tower_types(towers)
        print(f"[+] Tower types: {tower_types}")
        
        mapping = tool.map_tower_locations(towers)
        if mapping:
            print(f"[+] Coverage area: {mapping['estimated_area_sq_km']} km²")
        
        network_info = tool.extract_network_info(towers)
        if network_info['operators']:
            print(f"[+] Operators found: {list(network_info['operators'].keys())}")
        
        # Export
        print()
        tool.export_report(args.output)
        
        if args.csv:
            tool.export_csv(args.csv)
        
        # Summary
        print(tool.generate_summary())
    else:
        print("[-] No tower data retrieved. Check coordinates and API availability.")
        sys.exit(1)


if __name__ == '__main__':
    main()
