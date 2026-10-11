#!/usr/bin/env python3
import json, sqlite3, urllib.parse, mimetypes, os
from http.server import ThreadingHTTPServer, SimpleHTTPRequestHandler
ROOT=os.path.dirname(os.path.abspath(__file__))
DBS={'2019':os.path.join(ROOT,'top_scientists_2019.db'),'2021':os.path.join(ROOT,'top_scientists_2021.db'),'2020':os.path.join(ROOT,'top_scientists_2020.db'),'2022':os.path.join(ROOT,'top_scientists_2022.db'),'2023':os.path.join(ROOT,'top_scientists_2023.db'),'2024':os.path.join(ROOT,'top_scientists_2024.db'),'2025':os.path.join(ROOT,'top_scientists_2025.db')}
DEFAULT_YEAR='2025'
FIELDS=['authfull','inst_name','cntry','sm_field','sm_subfield_1','sm_subfield_2','rank','rank_ns','nc2525','nc2525_ns','h25','h25_ns','c','c_ns','np6025','firstyr','lastyr','selfpct','cosmo','top_cntry_career','top_cntry_share','cntry_cov']

import re
LOC_FILE=os.path.join(ROOT,'institution_locations.json')
_LOC=None
def norm(x):
 return re.sub(r'[^a-z0-9]+',' ',(x or '').lower()).strip()
def load_locations():
 global _LOC
 if _LOC is None:
  try:
   raw=json.load(open(LOC_FILE,encoding='utf-8')); _LOC={(x['country'].lower(),norm(x['institution'])):x for x in raw}
  except: _LOC={}
 return _LOC

def conn(year=DEFAULT_YEAR):
 p=DBS.get(str(year),DBS[DEFAULT_YEAR]); c=sqlite3.connect(p); c.row_factory=sqlite3.Row; return c

def filters(q):
 wh=[]; a=[]
 def eq(k,col=None):
  if q.get(k): wh.append(f'{col or k}=?'); a.append(q[k][0])
 def rng(k,col,op):
  if q.get(k) and q[k][0] != '':
   try: a.append(float(q[k][0])); wh.append(f'{col}{op}?')
   except: pass
 eq('country','cntry'); eq('field','sm_field'); eq('subfield','sm_subfield_1')
 if q.get('search'):
  s='%'+q['search'][0]+'%'; wh.append('(authfull LIKE ? OR inst_name LIKE ?)'); a += [s,s]
 for k,col,op in [('min_cites','nc2525','>='),('max_cites','nc2525','<='),('min_h','h25','>='),('max_rank','rank','<='),('min_papers','np6025','>='),('firstyr','firstyr','>='),('lastyr','lastyr','<=')]: rng(k,col,op)
 return (' WHERE '+' AND '.join(wh) if wh else ''),a

class H(SimpleHTTPRequestHandler):
 def translate_path(self,path):
  p=urllib.parse.urlparse(path).path
  return os.path.join(ROOT, 'index.html' if p=='/' else p.lstrip('/'))
 def sendj(self,obj):
  b=json.dumps(obj,ensure_ascii=False).encode(); self.send_response(200); self.send_header('Content-Type','application/json; charset=utf-8'); self.send_header('Content-Length',str(len(b))); self.end_headers(); self.wfile.write(b)
 def do_GET(self):
  u=urllib.parse.urlparse(self.path)
  if not u.path.startswith('/api/'): return super().do_GET()
  q=urllib.parse.parse_qs(u.query); year=q.get('year',[DEFAULT_YEAR])[0]; year=year if year in DBS else DEFAULT_YEAR; c=conn(year)
  try:
   if u.path=='/api/meta':
    out={}
    for key,col in [('countries','cntry'),('fields','sm_field'),('subfields','sm_subfield_1')]: out[key]=[r[0] for r in c.execute(f'SELECT DISTINCT {col} FROM scientists WHERE {col} IS NOT NULL ORDER BY {col}')]
    out['total']=c.execute('SELECT COUNT(*) FROM scientists').fetchone()[0]; out['year']=year; out['years']=sorted(DBS.keys(),reverse=True)
    out['ranges']=dict(c.execute('SELECT MIN(nc2525),MAX(nc2525),MIN(h25),MAX(h25),MIN(rank),MAX(rank),MIN(np6025),MAX(np6025) FROM scientists').fetchone()) if False else {}
    return self.sendj(out)
   wh,a=filters(q)
   if u.path=='/api/scientists':
    page=max(1,int(q.get('page',['1'])[0])); size=min(2000,max(10,int(q.get('size',['10'])[0]))); sort=q.get('sort',['rank'])[0]; sort=sort if sort in FIELDS else 'rank'; direction='DESC' if q.get('dir',['asc'])[0]=='desc' else 'ASC'
    total=c.execute('SELECT COUNT(*) FROM scientists'+wh,a).fetchone()[0]
    rows=[dict(r) for r in c.execute('SELECT id,'+','.join(FIELDS)+' FROM scientists'+wh+f' ORDER BY {sort} {direction} LIMIT ? OFFSET ?',a+[size,(page-1)*size])]
    return self.sendj({'total':total,'page':page,'size':size,'rows':rows})
   if u.path=='/api/map':
    qmap={k:v for k,v in q.items() if k!='country'}; whm,am=filters(qmap)
    rows=[dict(r) for r in c.execute('SELECT UPPER(cntry) country,COUNT(*) n,AVG(nc2525) avg_cites,MIN(rank) best_rank FROM scientists'+whm+' GROUP BY cntry ORDER BY n DESC',am)]
    return self.sendj(rows)
   if u.path=='/api/institution-scientists':
    institution=q.get('institution',[''])[0]
    if not institution: return self.sendj({'total':0,'rows':[]})
    clause=(' AND ' if wh else ' WHERE ')+'inst_name=?'; aa=a+[institution]
    total=c.execute('SELECT COUNT(*) FROM scientists'+wh+clause,aa).fetchone()[0]
    rows=[dict(r) for r in c.execute('SELECT id,'+','.join(FIELDS)+' FROM scientists'+wh+clause+' ORDER BY rank ASC LIMIT 5000',aa)]
    return self.sendj({'total':total,'rows':rows,'truncated':total>5000})
   if u.path=='/api/institutions':
    rows=[dict(r) for r in c.execute("SELECT inst_name institution,COUNT(*) n,AVG(nc2525) avg_cites,MIN(rank) best_rank FROM scientists"+wh+" AND inst_name IS NOT NULL AND TRIM(inst_name)<>'' GROUP BY inst_name ORDER BY n DESC LIMIT 30" if wh else "SELECT inst_name institution,COUNT(*) n,AVG(nc2525) avg_cites,MIN(rank) best_rank FROM scientists WHERE inst_name IS NOT NULL AND TRIM(inst_name)<>'' GROUP BY inst_name ORDER BY n DESC LIMIT 30",a)]
    locs=load_locations(); cc=q.get('country',[''])[0].lower()
    for r in rows:
     z=locs.get((cc,norm(r['institution'])))
     if z: r.update({'city':z.get('city'),'domain':z.get('domain',''),'location_source':z.get('source','')})
    return self.sendj(rows)
   if u.path=='/api/cities':
    # City coordinates are enriched from a curated institution-location registry plus conservative name matching.
    country=q.get('country',[''])[0].lower()
    if not country: return self.sendj([])
    wh,a=filters(q)
    rows=c.execute("SELECT inst_name,COUNT(*) n FROM scientists"+wh+" AND inst_name IS NOT NULL AND TRIM(inst_name)<>'' GROUP BY inst_name ORDER BY n DESC" if wh else "SELECT inst_name,COUNT(*) n FROM scientists WHERE inst_name IS NOT NULL AND TRIM(inst_name)<>'' GROUP BY inst_name ORDER BY n DESC",a).fetchall()
    locs=load_locations()
    agg={}; matched=0; total=sum(r['n'] for r in rows)
    for r in rows:
     z=locs.get((country,norm(r['inst_name'])))
     if not z: continue
     matched+=r['n']; key=(z['city'],z['lat'],z['lon'])
     agg[key]=agg.get(key,0)+r['n']
    out=[{'city':k[0],'lat':k[1],'lon':k[2],'n':n} for k,n in agg.items()]
    out.sort(key=lambda x:x['n'],reverse=True)
    return self.sendj({'rows':out,'matched_scientists':matched,'total_scientists':total,'coverage':(matched/total if total else 0)})
   if u.path.startswith('/api/scientist/'):
    sid=int(u.path.rsplit('/',1)[1]); r=c.execute('SELECT * FROM scientists WHERE id=?',(sid,)).fetchone(); return self.sendj(dict(r) if r else {})
   self.send_error(404)
  finally: c.close()
if __name__=='__main__':
 print('WoTTS: http://127.0.0.1:8000')
 ThreadingHTTPServer(('127.0.0.1',8000),H).serve_forever()
