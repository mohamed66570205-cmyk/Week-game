const http=require('http'),fs=require('fs'),path=require('path'),{WebSocketServer}=require('ws');
const html=fs.readFileSync(path.join(__dirname,'index.html'));
const srv=http.createServer((q,r)=>{r.writeHead(200,{'Content-Type':'text/html; charset=utf-8'});r.end(html)});
const wss=new WebSocketServer({server:srv,maxPayload:256*1024});
const rooms={};let n=0;
const send=(w,m)=>{if(w&&w.readyState===1)w.send(JSON.stringify(m))};
wss.on('connection',ws=>{
  ws.id='p'+(++n);
  ws.on('message',d=>{
    let m;try{m=JSON.parse(d)}catch(e){return}
    if(m.t==='create'){
      let c;do{c=String(1000+Math.floor(Math.random()*9000))}while(rooms[c]);
      rooms[c]={host:ws,cl:{}};ws.room=c;ws.isHost=true;send(ws,{t:'created',code:c});
    }else if(m.t==='join'){
      const R=rooms[String(m.code)];
      if(!R||Object.keys(R.cl).length>=7)return send(ws,{t:'err',msg:'الغرفة غير موجودة أو ممتلئة'});
      R.cl[ws.id]=ws;ws.room=String(m.code);send(ws,{t:'joined',id:ws.id});
      send(R.host,{t:'in',from:ws.id,name:m.name,av:m.av});
    }else{
      const R=rooms[ws.room];if(!R)return;
      if(ws.isHost){if(m.to)send(R.cl[m.to],m);else Object.values(R.cl).forEach(c=>send(c,m))}
      else{m.from=ws.id;send(R.host,m)}
    }
  });
  ws.on('close',()=>{
    const R=rooms[ws.room];if(!R)return;
    if(ws.isHost){Object.values(R.cl).forEach(c=>send(c,{t:'err',msg:'خرج صاحب الغرفة'}));delete rooms[ws.room]}
    else{delete R.cl[ws.id];send(R.host,{t:'left',from:ws.id})}
  });
});
srv.listen(process.env.PORT||3000,()=>console.log('Wink on port '+(process.env.PORT||3000)));
