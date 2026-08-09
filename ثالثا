// Cloud Functions: createShare و updateList
const functions = require('firebase-functions');
const admin = require('firebase-admin');
const crypto = require('crypto');
const cors = require('cors')({ origin: true });

admin.initializeApp();
const db = admin.firestore();

function makeToken() {
  return crypto.randomBytes(24).toString('hex'); // طول كافٍ للتخمين الصعب
}

exports.createShare = functions.https.onRequest(async (req, res) => {
  return cors(req, res, async () => {
    if (req.method !== 'POST') return res.status(405).json({ error: 'Method Not Allowed' });

    try {
      const body = req.body || {};
      const name = body.name || 'قائمة مؤقتات';
      const timers = Array.isArray(body.timers) ? body.timers : [];
      const expiresDays = Number(body.expiresDays || 30);
      const baseUrl = String(body.baseUrl || '').replace(/\/$/, '');

      // أنشئ المستند في القوائم
      const listRef = await db.collection('lists').add({
        name,
        data: timers,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });

      const id = listRef.id;
      const editToken = makeToken();
      const viewToken = makeToken();

      // خزّن التوكنات في مجموعة خاصة لا يمكن للعميل قراءتها
      await db.collection('tokens').doc(id).set({
        editToken,
        viewToken,
        expiresAt: admin.firestore.Timestamp.fromDate(new Date(Date.now() + expiresDays * 24 * 3600 * 1000))
      });

      const editLink = baseUrl ? `${baseUrl}?listId=${id}&token=${editToken}&mode=edit` : null;
      const viewLink = baseUrl ? `${baseUrl}?listId=${id}&token=${viewToken}&mode=view` : null;

      return res.json({ listId: id, editLink, viewLink });
    } catch (err) {
      console.error('createShare error', err);
      return res.status(500).json({ error: String(err) });
    }
  });
});

exports.updateList = functions.https.onRequest(async (req, res) => {
  return cors(req, res, async () => {
    if (req.method !== 'POST') return res.status(405).json({ error: 'Method Not Allowed' });

    try {
      const body = req.body || {};
      const listId = body.listId;
      const token = body.editToken;
      const newData = Array.isArray(body.timers) ? body.timers : null;
      const name = body.name;

      if (!listId || !token || newData === null) return res.status(400).json({ error: 'missing parameters' });

      const tokenDoc = await db.collection('tokens').doc(listId).get();
      if (!tokenDoc.exists) return res.status(403).json({ error: 'invalid list' });
      const tokenData = tokenDoc.data();
      if (!tokenData || tokenData.editToken !== token) return res.status(403).json({ error: 'invalid edit token' });

      // تحديث المستند بالقيم الجديدة
      const updatePayload = {
        data: newData,
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      };
      if (typeof name === 'string') updatePayload.name = name;

      await db.collection('lists').doc(listId).set(updatePayload, { merge: true });

      return res.json({ ok: true });
    } catch (err) {
      console.error('updateList error', err);
      return res.status(500).json({ error: String(err) });
    }
  });
});
