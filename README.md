// app/page.tsx
import { google } from "googleapis";
import { 
  MapPin, Building2, User, Phone, Calendar, Key, 
  Maximize2, FileText, FileSpreadsheet, Share2, MessageCircle, Copy 
} from "lucide-react";

// 日付フォーマットのヘルパー
function formatDate(dateVal: any): string {
  if (!dateVal) return "--";
  const d = new Date(dateVal);
  if (isNaN(d.getTime())) return String(dateVal);
  return `${d.getFullYear()}/${String(d.getMonth() + 1).padStart(2, '0')}/${String(d.getDate()).padStart(2, '0')}`;
}

export default async function ProjectPage({
  searchParams,
}: {
  searchParams: Promise<{ id?: string }>;
}) {
  const { id } = await searchParams;

  if (!id) {
    return <div className="p-8 text-center text-red-500 font-semibold">IDが指定されていません。</div>;
  }

  try {
    // 1. Google Sheets APIの認証設定（環境変数から取得）
    const auth = new google.auth.JWT(
      process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL,
      undefined,
      process.env.GOOGLE_PRIVATE_KEY?.replace(/\\n/g, "\n"),
      ["https://www.googleapis.com/auth/spreadsheets.readonly"]
    );

    const sheets = google.sheets({ version: "v4", auth });
    const spreadsheetId = process.env.SPREADSHEET_ID; // スプレッドシートID

    // 2. データの取得
    const response = await sheets.spreadsheets.values.get({
      spreadsheetId,
      range: "data!A1:Z1000", // 必要に応じて範囲を広げてください
    });

    const values = response.data.values;
    if (!values || values.length === 0) {
      return <div className="p-8 text-center text-red-500">シート「data」データが見つかりません。</div>;
    }

    const header = values[0];
    const dataRows = values.slice(1);

    const getIdx = (name: string) => {
      const index = header.indexOf(name);
      if (index === -1) throw new Error(`ヘッダー「${name}」が見つかりません`);
      return index;
    };

    // ヘッダーインデックス取得
    const idIdx = getIdx("No");
    const clientIdx = getIdx("発注元");
    const staffIdx = getIdx("担当者");
    const telIdx = getIdx("連絡先");
    const titleIdx = getIdx("案件名");
    const addressIdx = getIdx("住所");
    const areaIdx = getIdx("延床面積");
    const kBoxIdx = getIdx("キーBOX");
    const checkIdx = getIdx("検査日");
    const locIdx = getIdx("緯度経度");
    const noteIdx = getIdx("備考");
    const attachedIdx = getIdx("添付資料");

    const dIdx = [getIdx("施工日1"), getIdx("施工日2"), getIdx("施工日3"), getIdx("施工日4")];
    const wIdx = [getIdx("工事内容1"), getIdx("工事内容2"), getIdx("工事内容3"), getIdx("工事内容4")];

    // ID検索
    const row = dataRows.find((r) => String(r[idIdx]) === String(id));
    if (!row) {
      return <div className="p-8 text-center text-red-500">ID「{id}」が見つかりません。</div>;
    }

    // データ抽出
    const title = row[titleIdx] || "無題の案件";
    const address = row[addressIdx] || "--";
    const client = row[clientIdx] || "--";
    const staff = row[staffIdx] || "--";
    const tel = row[telIdx] || "--";
    const check = formatDate(row[checkIdx]);
    const kBox = row[kBoxIdx] || "--";
    const area = row[areaIdx] || "--";
    const loc = row[locIdx] || "";
    const note = row[noteIdx] || "--";
    const attached = row[attachedIdx] || "";

    // GoogleマップURL
    const mapUrl = loc ? `https://www.google.com/maps/search/?api=1&query=${encodeURIComponent(loc)}` : null;

    // 施工予定のパース
    const schedules = [];
    for (let i = 0; i < 4; i++) {
      if (row[dIdx[i]] || row[wIdx[i]]) {
        schedules.push({
          date: formatDate(row[dIdx[i]]),
          content: row[wIdx[i]] || "内容なし",
        });
      }
    }

    // 共有用URL（Vercelのホスト名 or リクエストURL）
    // 本番環境のURLを環境変数として定義しておくと安全です
    const baseUrl = process.env.NEXT_PUBLIC_APP_URL || "https://your-vercel-domain.vercel.app";
    const shareUrl = `${baseUrl}?id=${encodeURIComponent(id)}`;
    const shareMessage = `【案件共有】\n${title}\n${address}\n${shareUrl}`;
    const lineUrl = `https://line.me/R/msg/text/?${encodeURIComponent(shareMessage)}`;

    return (
      <main className="min-h-screen bg-slate-50 dark:bg-slate-900 py-6 px-4 sm:px-6 lg:px-8">
        <div className="max-w-md mx-auto space-y-4">
          
          {/* メインカード */}
          <div className="bg-white dark:bg-slate-800 rounded-2xl shadow-md border border-slate-100 dark:border-slate-700 overflow-hidden">
            <div className="p-6 bg-gradient-to-r from-emerald-500 to-teal-600 text-white">
              <span className="text-xs font-semibold bg-white/20 px-2 py-1 rounded-full uppercase tracking-wider">No. {id}</span>
              <h1 className="text-xl font-bold mt-2 tracking-tight">{title}</h1>
            </div>

            <div className="p-6 space-y-4">
              {/* 各種情報リスト */}
              <div className="grid grid-cols-[24px_1fr] gap-x-3 gap-y-4 text-sm text-slate-600 dark:text-slate-300">
                <MapPin className="text-slate-400 w-5 h-5" />
                <div>
                  <div className="text-xs text-slate-400 font-medium">住所</div>
                  <div className="font-medium text-slate-800 dark:text-white mt-0.5">{address}</div>
                </div>

                <Building2 className="text-slate-400 w-5 h-5" />
                <div>
                  <div className="text-xs text-slate-400 font-medium">発注元 / 担当者</div>
                  <div className="font-medium text-slate-800 dark:text-white mt-0.5">{client} ({staff} 様)</div>
                </div>

                <Phone className="text-slate-400 w-5 h-5" />
                <div>
                  <div className="text-xs text-slate-400 font-medium">連絡先</div>
                  <div className="font-medium text-slate-800 dark:text-white mt-0.5">{tel}</div>
                </div>

                <Calendar className="text-slate-400 w-5 h-5" />
                <div>
                  <div className="text-xs text-slate-400 font-medium">検査日</div>
                  <div className="font-medium text-slate-800 dark:text-white mt-0.5">{check}</div>
                </div>

                <Key className="text-slate-400 w-5 h-5" />
                <div>
                  <div className="text-xs text-slate-400 font-medium">キーBOX</div>
                  <div className="font-medium text-slate-800 dark:text-white mt-0.5">{kBox}</div>
                </div>

                <Maximize2 className="text-slate-400 w-5 h-5" />
                <div>
                  <div className="text-xs text-slate-400 font-medium">延床面積</div>
                  <div className="font-medium text-slate-800 dark:text-white mt-0.5">{area} ㎡</div>
                </div>

                <FileText className="text-slate-400 w-5 h-5" />
                <div>
                  <div className="text-xs text-slate-400 font-medium">備考</div>
                  <div className="font-medium text-slate-800 dark:text-white mt-0.5 whitespace-pre-wrap">{note}</div>
                </div>
              </div>

              {/* 添付資料がある場合 */}
              {attached && (
                <div className="pt-2">
                  <a href={attached} target="_blank" rel="noopener noreferrer" className="flex items-center justify-center gap-2 w-full py-2.5 px-4 bg-sky-50 dark:bg-sky-950/30 text-sky-600 dark:text-sky-400 rounded-xl font-medium text-sm hover:bg-sky-100 dark:hover:bg-sky-950/50 transition">
                    <FileSpreadsheet className="w-4 h-4" /> 参考資料を参照する
                  </a>
                </div>
              )}

              <hr className="border-slate-100 dark:border-slate-700" />

              {/* 施工予定セクション */}
              <div>
                <h2 className="text-sm font-bold text-slate-800 dark:text-white mb-2.5 flex items-center gap-1.5">👷 施工予定</h2>
                {schedules.length > 0 ? (
                  <div className="space-y-2">
                    {schedules.map((item, idx) => (
                      <div key={idx} className="p-3 bg-slate-50 dark:bg-slate-900 rounded-xl border border-slate-100 dark:border-slate-800 text-sm">
                        <span className="inline-block text-xs font-semibold text-teal-600 dark:text-teal-400 mb-0.5">{item.date}</span>
                        <div className="text-slate-700 dark:text-slate-200">{item.content}</div>
                      </div>
                    ))}
                  </div>
                ) : (
                  <div className="text-sm text-slate-400 italic text-center py-2">予定はありません</div>
                )}
              </div>

              {mapUrl && (
                <>
                  <hr className="border-slate-100 dark:border-slate-700" />
                  <a href={mapUrl} target="_blank" rel="noopener noreferrer" className="flex items-center justify-center gap-2 w-full py-3 bg-blue-600 hover:bg-blue-700 text-white font-medium rounded-xl text-sm transition shadow-sm shadow-blue-500/10">
                    📍 Googleマップを開く
                  </a>
                </>
              )}
            </div>
          </div>

          {/* クライアントコンポーネント（ボタンアクション用） */}
          <ActionButtons lineUrl={lineUrl} shareUrl={shareUrl} shareMessage={shareMessage} />

        </div>
      </main>
    );
  } catch (error: any) {
    return (
      <div className="p-8 max-w-md mx-auto text-center">
        <div className="bg-red-50 dark:bg-red-950/30 text-red-600 dark:text-red-400 p-4 rounded-xl border border-red-100 dark:border-red-900 text-sm">
          <p className="font-bold">エラーが発生しました</p>
          <p className="text-xs mt-1 text-left whitespace-pre-wrap">{error.message}</p>
        </div>
      </div>
    );
  }
}
